# 03 · FreeRTOS

> **Phase 7 — Edge and IoT** · Lesson 03 of 09
> ⬅️ Previous: [AWS IoT Greengrass](./02-aws-iot-greengrass.md) · ➡️ Next: [Amazon Snow Family](./04-aws-snow-family.md) · 🏠 [Phase overview](./README.md)

## Learning Objectives

By the end of this lesson you will be able to:

- Explain what the FreeRTOS kernel provides and what it deliberately does not
- Assemble a secure MQTT client on a microcontroller from the AWS IoT Device SDK for Embedded C libraries
- Estimate the flash and RAM cost of TLS plus MQTT plus the OTA agent on a Cortex-M class device
- Wire up an over-the-air firmware update using AWS IoT Jobs, AWS IoT streams, and code signing
- Pick a demo board and configuration for a real project instead of starting from scratch
- Reason about long-term support and library pinning for devices that ship for a decade

**Estimated time:** 90 min · **Difficulty:** Advanced

---

## Core Concepts

**FreeRTOS** is a real-time kernel, not a full operating system. It gives you a preemptive scheduler, tasks, queues, semaphores, mutexes, software timers, event groups, stream buffers, and message buffers — and then it stops. There is no memory protection, no filesystem, no network stack, no user accounts. Everything above that line is your responsibility, and every layer you add costs kilobytes of RAM you may not have. It ships under the MIT license and runs on dozens of microcontroller architectures, which is why it became the default choice for battery-powered sensors and industrial controllers.

What makes FreeRTOS interesting in this roadmap is the second half of the name: the AWS IoT libraries that run on top of it. The **AWS IoT Device SDK for Embedded C** is a set of small, dependency-light C libraries, each one solving exactly one problem:

- **coreMQTT** — the MQTT 3.1.1 client. No dynamic allocation, no threading assumptions, caller-provided buffers.
- **coreHTTP** — an HTTP/1.1 client for things that cannot justify MQTT, or for fetching firmware directly.
- **coreJSON** and **corePKCS11** — a parser for the small JSON documents you actually receive, and a PKCS#11 abstraction so keys can live in secure hardware rather than in flash.
- **backoffAlgorithm** and **coreSNTP** — retry jitter, and the time synchronization that certificate validity checks depend on.
- **device shadow, device defender, jobs, and fleet provisioning libraries** — the client side of the AWS IoT services from Lesson 01, sized for constrained devices.
- **ota-for-aws-iot-embedded-sdk** — the OTA agent, which downloads a signed image in blocks over MQTT or HTTP and hands it to your bootloader.

The design philosophy is worth internalizing because it explains the API shape: **no hidden allocation, no blocking calls, no global state, no threading model imposed on you.** Every library takes explicit buffers and a transport interface you implement. That is what makes the same code run on a 64 KB Cortex-M0 and a 512 KB Cortex-M4 without changes beyond configuration.

TLS is where the memory goes. A minimal secure MQTT stack needs a TLS implementation — **mbedTLS** is the common choice, with **wolfSSL** as a lower-footprint alternative for constrained targets — plus certificate storage, plus receive and transmit buffers. People are consistently surprised that the cryptography, not the protocol, dominates: coreMQTT itself can run in single-digit kilobytes, while the TLS handshake and certificate parsing can be five to ten times that.

Finally, **over-the-air updates** are the thing that makes a deployed fleet manageable, and the FreeRTOS OTA agent is the reference implementation. The agent receives a job document, requests the image in blocks from an AWS IoT stream, writes each block through your `writeBlock` callback into bootloader-managed flash, verifies the cryptographic signature using a public key you provisioned, sets the new image as the boot target, and reboots. If the image fails to boot twice, the bootloader rolls back. You do not get to skip the signature: an unsigned OTA channel is a remote code execution channel into a device inside a customer's building.

### Key terminology

| Term | Meaning |
|------|---------|
| **Kernel** | FreeRTOS scheduler and primitives from `FreeRTOS-Kernel`. MIT licensed, ~6–12 KB of code depending on configuration. |
| **coreMQTT** | MQTT client library. Application supplies network and time functions via a transport interface. |
| **Transport interface** | Your callbacks — `send`, `recv`, plus optional time functions — that connect a library to your TLS stack. |
| **PKCS#11** | Interface used by corePKCS11 so private keys can stay in a secure element or TrustZone partition. |
| **OTA agent** | `ota-for-aws-iot-embedded-sdk`; downloads, verifies, and stages firmware images. |
| **Job document** | The JSON delivered by AWS IoT Jobs; for OTA it contains an `afr_ota` block describing the image. |
| **AWS IoT stream** | A service object that serves firmware data to devices, chunked and resumable, over MQTT or HTTP. |
| **Code signing** | An AWS Signer job that produces a signed image and a signature file the device verifies. |
| **LTS release** | A pinned set of kernel plus libraries with a multi-year support window, e.g. `202210.00 LTS`. |

---

## Architecture Diagram

What lives inside the microcontroller, and what crosses the wire:

```mermaid
flowchart LR
    subgraph mcu["Microcontroller (e.g. Cortex-M4, 256 KB flash)"]
        app["Application tasks"]
        kernel["FreeRTOS kernel (tasks, queues)"]
        mqtt["coreMQTT"]
        http["coreHTTP"]
        ota["OTA agent"]
        pkcs["corePKCS11"]
        tls["mbedTLS / wolfSSL"]
        flash["Flash: firmware + bootloader"]
    end
    app --> kernel
    app --> mqtt
    app --> ota
    mqtt --> tls
    http --> tls
    ota --> http
    ota --> flash
    tls --> pkcs
    tls -->|"TCP 8883 mutual TLS"| iot["AWS IoT Core"]
    ota -->|"job document (MQTT)"| jobs["AWS IoT Jobs"]
    ota -->|"image blocks"| stream["AWS IoT stream (S3 backed)"]
```

The OTA path from your build machine to a device in the field:

```mermaid
sequenceDiagram
    participant Dev as Developer / CI
    participant S3 as Amazon S3
    participant Signer as AWS Signer
    participant IoT as AWS IoT Jobs
    participant D as Device (OTA agent)
    Dev->>S3: Upload unsigned firmware image
    Dev->>Signer: start-signing-job (AmazonFreeRTOS-Default profile)
    Signer->>S3: Write signed image + signature
    Dev->>IoT: create-stream + create-ota-update
    IoT->>D: Job document with afr_ota block
    D->>IoT: Request file blocks
    IoT-->>D: Signed image blocks
    D->>D: Verify signature, write to inactive bank
    D->>D: Set boot target and reboot
    alt Verification or boot fails twice
        D->>D: Roll back to previous image
    end
    D-->>IoT: Job execution SUCCEEDED or FAILED
```

OTA agent states — the sequence you watch when an update stalls:

```mermaid
stateDiagram-v2
    [*] --> Ready
    Ready --> RequestJobDocument
    RequestJobDocument --> WaitingForJobDocument
    WaitingForJobDocument --> CreatingFile
    CreatingFile --> RequestFileBlock
    RequestFileBlock --> WaitingForFileBlock
    WaitingForFileBlock --> RequestFileBlock
    WaitingForFileBlock --> ClosingFile
    ClosingFile --> WaitingForJobCleanup
    WaitingForJobCleanup --> Ready
    RequestJobDocument --> Failed
    WaitingForJobDocument --> Failed
    WaitingForFileBlock --> Failed
    Failed --> [*]
```

---

## Hands-on Lab: Build, Sign, and Deliver a Firmware Update

**Goal:** Build the FreeRTOS coreMQTT mutual-auth demo, publish a new version, sign it, and deliver it as an OTA job to a registered thing.

**Prerequisites:** A host with `git`, `cmake` 3.13+, and a cross-compiler or a supported board toolchain. An AWS account with permissions for `s3`, `iot`, `signer`, and `iam:PassRole`. A registered thing with a certificate from Lesson 01 (or reuse the demo's provisioning path).

### Step 1 — Get the source and pin an LTS release

```bash
# The LTS branch bundles a kernel plus library versions that were tested together
git clone --recurse-submodules https://github.com/FreeRTOS/FreeRTOS.git --branch main freertos
cd freertos

# Inspect which libraries the embedded C SDK actually pulls in
ls FreeRTOS-Plus/Source/AWS
```

If you are starting a shipping product rather than an experiment, take the LTS bundle instead of `main` and record the exact commit hashes in your build pipeline. Devices live longer than your dependency-upgrade enthusiasm.

### Step 2 — Configure credentials and the broker endpoint

```c
/* demos/<board>/config_files/aws_clientcredential.h */
#define clientcredentialMQTT_BROKER_ENDPOINT "abcdefghijklmnop-ats.iot.eu-west-1.amazonaws.com"
#define clientcredentialIOT_THING_NAME       "sensor-0001"
```

```c
/* demos/<board>/config_files/aws_clientcredential_keys.h
 * Development only. Production devices must provision unique credentials
 * (see fleet provisioning in Lesson 01) instead of a shared demo key.
 */
#define clientcredentialCLIENT_CERTIFICATE_PEM "-----BEGIN CERTIFICATE-----\n..."
#define clientcredentialCLIENT_PRIVATE_KEY_PEM "-----BEGIN RSA PRIVATE KEY-----\n..."
```

```c
/* demos/<board>/config_files/core_mqtt_config.h
 * Right-size the buffers: every byte here is RAM that has to exist at boot.
 */
#define demosMQTT_TASK_STACK_SIZE  4096
#define demosMQTT_BUFFER_SIZE      2048
#define demosNETWORK_BUFFER_SIZE   2560
```

### Step 3 — Build the demo

```bash
# Board and vendor identifiers come from the demos/ directory layout
cmake -DVENDOR=espressif \
      -DBOARD=esp32_c3_devkitm_1 \
      -DCOMPILER=xtensa-esp32s3 \
      -S . -B build
cmake --build build
```

The demo publishes to the same MQTT topics your Lesson 01 rule already listens on, which is the point: a microcontroller is just another client of the same broker.

### Step 4 — Produce and sign a firmware image

```bash
# Pick a version string; the device compares it against the running version
export FW_VERSION=1.4.3
cp build/demo_image.bin "app-$FW_VERSION.bin"

aws s3api create-bucket --bucket iot-firmware-eu-west-1 --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1
aws s3api put-public-access-block --bucket iot-firmware-eu-west-1 \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-object --bucket iot-firmware-eu-west-1 --key "app-$FW_VERSION.bin" --body "app-$FW_VERSION.bin"
```

```bash
# A signing profile holds the certificate whose private key signs images.
# The device only needs the matching PUBLIC key, which is far safer to ship.
aws signer put-signing-profile \
  --profile-name FreeRTOSFirmwareProfile \
  --platform AmazonFreeRTOS-Default \
  --signing-material certificateArn=arn:aws:acm:eu-west-1:123456789012:certificate/11111111-2222-3333-4444-555555555555

# Sign the image; the output lands in S3 alongside the unsigned artifact
aws signer start-signing-job \
  --source 's3={bucketName=iot-firmware-eu-west-1,key=app-1.4.3.bin,version=abc123}' \
  --destination 's3={bucketName=iot-firmware-eu-west-1,prefix=signed/}' \
  --profile-name FreeRTOSFirmwareProfile \
  --client-request-token fw-143-signing
```

### Step 5 — Create the stream and the OTA update

```bash
# A stream is the resumable transport for image blocks
aws iot create-stream \
  --stream-id firmware-1-4-3 \
  --description "Firmware 1.4.3 for the sensor fleet" \
  --files fileId=0,s3Location={bucket=iot-firmware-eu-west-1,key=signed/app-1.4.3.bin,version=signedVersionId} \
  --role-arn arn:aws:iam::123456789012:role/IoTFirmwareStreamRole

# This single call creates the job, the job document, and the rollout config
aws iot create-ota-update \
  --ota-update-id firmware-1-4-3 \
  --description "Sensor firmware 1.4.3" \
  --targets arn:aws:iot:eu-west-1:123456789012:thing/sensor-0001 \
  --protocols MQTT \
  --target-selection-config '{"continueOnFailure":false}' \
  --files file://ota-files.json \
  --role-arn arn:aws:iam::123456789012:role/IoTOTAUpdateRole
```

```json
[
  {
    "fileName": "app-1.4.3.bin",
    "fileVersion": "1.4.3",
    "fileLocation": {
      "s3Location": {
        "bucket": "iot-firmware-eu-west-1",
        "key": "signed/app-1.4.3.bin",
        "version": "signedVersionId"
      }
    },
    "codeSigning": {
      "awsSignerJobId": "7c1f6c0a-1111-2222-3333-abcdefabcdef",
      "startSigningJobParameter": {
        "signingProfileName": "FreeRTOSFirmwareProfile"
      }
    }
  }
]
```

### Step 6 — Verify the rollout

```bash
# Which devices have taken the update, and which are still pending?
aws iot list-job-executions-for-job \
  --job-id "firmware-1-4-3" \
  --query 'executionSummaries[].{device:thingArn,status:executionSummary.status}'

# The device-side view: the OTA agent logs the block requests it makes
# [OTA] Requesting file block 17 of 63
# [OTA] Signature verification succeeded
```

Success is a job execution in `SUCCEEDED` state with no manual intervention on the device, and a second boot that reports the new firmware version in its shadow `reported` state.

---

## CLI Reference

```bash
# Register the very first firmware image as open (covered by code signing instead)
aws signer list-signing-profiles --include-canceled

# Track a signing job while the CI pipeline waits on it
aws signer describe-signing-job --job-id 7c1f6c0a-1111-2222-3333-abcdefabcdef

# Stop an in-flight OTA rollout across the fleet
aws iot cancel-job --job-id firmware-1-4-3 --force true

# Inspect the stream a device will download from
aws iot describe-stream --stream-id firmware-1-4-3

# Suspend and resume individual executions on problem devices
aws iot describe-job-execution --job-id firmware-1-4-3 --thing-name sensor-0001

# Read the device's reported firmware version from its shadow
aws iot-data get-thing-shadow --thing-name sensor-0001 --output text --query 'state.reported.fw'
```

| Command | Purpose |
|---------|---------|
| `signer start-signing-job` | Signs the image so devices can verify it before flashing |
| `iot create-stream` | Turns an S3 object into a resumable block stream for devices |
| `iot create-ota-update` | Creates the job, document, and rollout config in one call |
| `iot list-job-executions-for-job` | Fleet-wide progress view of the update |
| `iot cancel-job` | Emergency stop for a bad image |
| `iot describe-job-execution` | Per-device detail when one board disagrees with the fleet |

---

## Best Practices

| Practice | Why it matters |
|----------|----------------|
| Pin to an LTS release and record commit hashes | A device in a wall panel cannot absorb a surprise library upgrade |
| Right-size buffers from measured usage, not guesses | Every buffer byte is RAM; over-provisioning forces a bigger, costlier MCU |
| Never ship a development key in production firmware | One shared private key means one flash dump compromises the fleet |
| Put private keys in a secure element via corePKCS11 | Makes key extraction from a stolen device materially harder for attackers |
| Enable backoff with jitter on reconnects | A plant-wide power restore otherwise creates a reconnect storm against the broker |
| Design the bootloader with A/B banks and rollback | The only real protection against a bad OTA image |
| Verify the signature *before* writing to the boot bank | A partially written, unverified image is worse than no update |
| Budget time sync deliberately | Certificate validity checks fail on devices with a wrong RTC |

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| TLS buffer too small for the certificate chain | Handshake fails with a memory error on some devices only | Include room for the full chain plus record overhead, and test with the real chain |
| Wrong device clock | TLS fails with a certificate-not-yet-valid error | Sync time with coreSNTP before the first connection |
| Shared demo credentials in a shipped product | Certificate revocation bricks the fleet | Provision per-device certificates; the demo key is for the lab |
| OTA job targeted at a thing group with `continueOnFailure: false` | One bad board stalls the whole rollout | Use staged rollout configuration and a canary group instead |
| Firmware image larger than the receiving bank | Update downloads fully, then fails to boot | Size the partitions and check image size in the job's `filesize` field |
| Trusting `main` branch libraries | Behavior changes appear between builds with no change to your code | Pin versions; upgrade deliberately |
| Stack overflow in a TLS task | Random resets that look like hardware failure | Increase the task stack and enable `configCHECK_FOR_STACK_OVERFLOW` |
| No watchdog on the OTA path | Device bricks during a failed update | Use a hardware watchdog and a rollback-capable bootloader |

---

## Cost Considerations

There is no charge for FreeRTOS or the embedded C SDK — they are open source and free to ship in commercial products. The costs are the per-device AWS services those libraries call, and the much larger non-AWS costs of the firmware lifecycle.

On the AWS side, each device pays IoT Core connectivity and messaging charges. A sensor publishing a 500-byte message every minute is roughly 43,000 messages a month, which is well within the free tier for a small pilot but becomes the dominant line item at thousands of units. Firmware updates are where the message math changes: a 1 MB image is about 200 blocks of 5 KB, charged as messages, so a fleet-wide update with a 1 MB image across 5,000 devices costs proportionally to the fleet size — and doing it twice costs twice as much. Keep the configurable OTA block size sensible and use HTTP for large images, which avoids MQTT messaging charges.

The costs people forget are the physical ones: an MCU with enough flash and RAM for TLS plus OTA can be two to three times the price of one that cannot, and a fleet that needs a truck roll to recover from a bad image costs more in one incident than a decade of AWS messaging. Budget for A/B banks and a rollback-capable bootloader at design time.

---

## Security Considerations

- **Assume the device will be physically attacked.** Anything in flash can be read with a debug probe. Use a secure element or TrustZone-backed keystore for private keys, and enable read-out protection — then verify it is actually on in production units.
- **Randomness quality breaks cryptography.** A device with a poor entropy source produces predictable keys. Sources like anotherwise-good TRNG with insufficient seeding are a classic vulnerability; validate the entropy source before shipping.
- **Verify before you write.** Check the AWS Signer signature against the provisioned public key before the image touches the boot bank, and refuse unsigned images silently — never fall back to "update anyway".
- **Keep the private key on the signing side only.** The device holds the public key. If the signing key leaks, an attacker can push arbitrary firmware; keep it in AWS Signer or a KMS-backed HSM, never on a laptop or in CI variables.
- **Rotate and revoke at the fleet level.** A per-device certificate means you can revoke one stolen board. Plan for it: know how a technician in the field gets a replacement certificate without shipping the unit back.
- **Encrypt what leaves the device.** TLS protects telemetry in transit; do not let a "smaller footprint" argument justify plaintext MQTT on port 1883 in a production plant.
- **Watch the debug interface.** JTAG and SWD left enabled on shipped hardware is an extraction path for firmware and keys alike.
- **Log security events where you can see them.** Device Defender's embedded client reports metrics from the device side, which is the only way to notice a device behaving strangely from inside the plant network.

---

## Exercise

Take the demo you built and add a second task that samples a sensor represented by an analog input (or a simulated counter), publishes it on a five-second period, and participates in the shadow protocol: report the sampling interval in `reported`, honour a `desired` change from the cloud, and publish a delta acknowledgment.

Then estimate your real footprint: use the linker map file to report the flash consumed by kernel, mbedTLS, coreMQTT, and the OTA agent separately, and the static RAM each requires. Decide whether the device could support an HTTP-based OTA path at the same time as MQTT, and justify the answer with the numbers.

Success looks like: a device that changes its sampling rate from the console within one publish cycle, plus a table of flash and RAM per library with the total compared against your target MCU's datasheet limits.

---

## Resources

- [FreeRTOS — AWS IoT Libraries](https://www.freertos.org/iot-freertos/index.html)
- [FreeRTOS Kernel Documentation](https://www.freertos.org/Documentation/RTOS_book.html)
- [AWS IoT Device SDK for Embedded C](https://github.com/aws/aws-iot-device-sdk-embedded-C)
- [FreeRTOS LTS Libraries](https://github.com/FreeRTOS/FreeRTOS-LTS)
- [coreMQTT Library](https://www.freertos.org/Documentation/02-Kernel/03-Supported-devices/02-Libraries/coreMQTT)
- [FreeRTOS Over-the-Air Updates](https://docs.aws.amazon.com/freertos/latest/userguide/freertos-ota-dev.html)
- [AWS Signer Developer Guide](https://docs.aws.amazon.com/signer/latest/developerguide/Welcome.html)
- [AWS IoT Jobs](https://docs.aws.amazon.com/iot/latest/developerguide/iot-jobs.html)

---

## Next Steps

You now have devices talking, gateways running code locally, and microcontrollers that can update themselves. The next lesson changes scale completely: instead of kilobytes over MQTT, it moves petabytes by truck. The Snow family is what you use when the network is the bottleneck and no amount of protocol tuning fixes a 100 TB upload over a 100 Mbps line.

> ⬅️ Previous: [AWS IoT Greengrass](./02-aws-iot-greengrass.md) · ➡️ Next: [Amazon Snow Family](./04-aws-snow-family.md) · 🏠 [Phase overview](./README.md)
