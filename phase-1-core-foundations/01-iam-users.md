# 01 · IAM Users

> **Phase 1 — Core Foundations** · Lesson 01 of 18
> ➡️ Next: [IAM Roles](./02-iam-roles.md) · 🏠 [Phase overview](./README.md)

## Learning Objectives

By the end of this lesson you will be able to:

- Explain what an IAM user is and when it is the right identity type — and when it is not
- Create a user with console access, programmatic access, or both
- Assign permissions through groups rather than direct attachment
- Apply the credential-hygiene practices that stop the most common AWS account compromise

**Estimated time:** 45 min · **Difficulty:** Beginner

---

## Core Concepts

An **IAM user** is a digital identity in AWS that represents a person or an application. It authenticates to AWS and then acts with whatever permissions its policies grant.

Users split into two broad categories, and the distinction matters because the security advice differs:

- **Human users** — developers, administrators, operators. They typically need console access, MFA, and short-lived sessions.
- **Programmatic users** — applications, CI/CD pipelines, automation scripts. They need access keys, and they are the category you should be actively trying to eliminate.

Each user carries two things:

- **Credentials** — a username and password for the console, and/or access keys for the API, CLI, and SDKs
- **Permissions** — granted through IAM policies, ideally attached to a group rather than to the user directly

The single most important thing to internalize in this lesson is that **IAM users with access keys are a legacy pattern**. Every long-lived access key is a credential that can leak, must be rotated, and cannot be scoped to a single job. The modern alternatives — IAM roles for compute, IAM Identity Center for humans, OIDC federation for CI/CD — remove the long-lived secret entirely. Learn users because they still exist everywhere in real accounts and because they are a common exam topic, but reach for them last.

### Key terminology

| Term | Meaning |
|------|---------|
| **IAM user** | A long-lived identity with permanent credentials. |
| **Access key** | An ID and secret pair used for programmatic access. Long-lived by default. |
| **Console password** | The credential used to sign in to the AWS Management Console. |
| **IAM group** | A collection of users that policies can be attached to. Users inherit the group's permissions. |
| **Inline policy** | A policy embedded in a single user or group; not reusable. |
| **Managed policy** | A standalone policy that can be attached to many identities. |
| **Account alias** | A friendly sign-in URL, e.g. `https://my-company.signin.aws.amazon.com/console`. |
| **`AmazonS3ReadOnlyAccess`** | An AWS-managed policy granting read-only S3 access. |

---

## Architecture Diagram

How a user's credentials turn into an authorized API call:

```mermaid
flowchart LR
    human["Human user"] -->|"console password + MFA"| console["AWS Management Console"]
    app["Application or CI job"] -->|"access key ID + secret"| api["AWS API / CLI / SDK"]
    console --> sts["Identity resolves to a principal ARN"]
    api --> sts
    sts --> eval{"Policy evaluation"}
    grp["Group policies"] --> eval
    inline["User inline policies"] --> eval
    direct["Directly attached managed policies"] --> eval
    eval -->|"allowed"| svc["Service API call succeeds"]
    eval -->|"denied"| err["AccessDenied"]
```

The two credential paths, and why the right-hand one is being phased out:

```mermaid
flowchart TD
    need{"Does a human need access?"}
    need -->|"Yes"| ic["IAM Identity Center\n(federated, MFA, temporary)"]
    need -->|"No, it is a workload"| wl{"What kind of workload?"}
    wl -->|"AWS compute"| role["IAM role\n(automatic rotation)"]
    wl -->|"External CI/CD"| oidc["OIDC federation\n(no stored secret)"]
    wl -->|"Genuinely cannot federate"| key["IAM user with access key\nLAST RESORT"]
```

---

## Hands-on Lab: Creating an IAM User

**Goal:** Create a user with a group-based permission model, and verify that the permissions are exactly what you intended — no more.

**Prerequisites:** An AWS account with administrative access, and the AWS CLI configured.

### Step 1 — Open IAM

Go to the [IAM Console](https://console.aws.amazon.com/iam/), or work entirely from the CLI as shown below.

### Step 2 — Create a group first

Groups come first because attaching permissions to a group and then adding users to it is the manageable pattern. A user created without a group tends to accumulate ad-hoc policies that nobody can audit.

```bash
# Create a group that will hold the read-only S3 permission
aws iam create-group --group-name Developers

# Attach a managed policy to the group, not to the user
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 3 — Create the user

```bash
# Create the user without any credentials first
aws iam create-user --user-name dev-user \
  --tags Key=Owner,Value=platform-team Key=Purpose,Value=lab

# Add the user to the group so it inherits the group's permissions
aws iam add-user-to-group --group-name Developers --user-name dev-user
```

Creating the user and granting credentials are deliberately separate steps. This lets you set a password policy and MFA requirement before any credential exists.

### Step 4 — Grant credentials

```bash
# Set an account alias so the sign-in URL is memorable
aws iam create-account-alias --account-alias my-company-name

# Console access: the user must change the password at first sign-in
aws iam create-login-profile \
  --user-name dev-user \
  --password 'REPLACE_ME_Min12Chars!' \
  --password-reset-required

# Programmatic access — only if genuinely needed
aws iam create-access-key --user-name dev-user
```

The `--password-reset-required` flag matters: without it, whoever created the account knows the user's password indefinitely.

### Step 5 — Require MFA

Creating a console user without MFA leaves the account one leaked password away from compromise.

```bash
# Force the user to set up MFA on first sign-in by scoping their permissions
cat > require-mfa.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEverythingExceptMFA",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
      }
    }
  ]
}
EOF

aws iam put-user-policy \
  --user-name dev-user \
  --policy-name RequireMFA \
  --policy-document file://require-mfa.json
```

This denies everything until MFA is present. The `BoolIfExists` is important — it lets the user's own MFA-enrolment API calls through, which a plain `Bool` would block, locking the user out permanently.

### Step 6 — Verify

```bash
# Confirm the user's effective permission path: via exactly one group
aws iam list-groups-for-user --user-name dev-user
aws iam list-attached-group-policies --group-name Developers
aws iam list-user-policies --user-name dev-user

# Confirm the user can read S3 and cannot write to it
aws s3 ls --profile dev-user
aws s3 cp /etc/hosts s3://some-bucket/ --profile dev-user   # expected: AccessDenied
```

If the copy succeeds, the user has more than the group grants — check for a directly attached policy you did not expect.

---

## CLI Reference

```bash
# Identity check — the fastest way to know who you are authenticated as
aws sts get-caller-identity

# List users and flag any with console access
aws iam list-users --query 'Users[].[UserName,PasswordLastUsed]' --output table

# Download the credential report — the audit view of every user in the account
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d > creds.csv
# Columns of interest: password_enabled, mfa_active, access_key_1_active, access_key_1_last_rotated

# List every user with an active access key, and when that key was last rotated.
# Note: report generation is asynchronous — wait before reading it back.
aws iam generate-credential-report >/dev/null && sleep 5
aws iam get-credential-report --query 'Content' --output text | base64 -d | \
  awk -F, 'NR==1{next} $9=="true" {print "user="$1, "key1_rotated="$10}'

# Deactivate a key immediately if it leaks, without deleting it (reversible step)
aws iam update-access-key --user-name dev-user \
  --access-key-id AKIAIOSFODNN7EXAMPLE --status Inactive

# List every policy that applies to a user, in one place
aws iam list-attached-user-policies --user-name dev-user
aws iam list-user-policies --user-name dev-user
```

| Command | Purpose |
|---------|---------|
| `get-caller-identity` | Confirms which principal you are using — first step in any auth bug |
| `generate-credential-report` | The account-wide audit of passwords, MFA, and key ages |
| `update-access-key --status Inactive` | Disables a key reversibly, the correct first response to a leak |
| `list-groups-for-user` | Confirms permissions arrive via a group rather than ad hoc |

---

## Best Practices

| Practice | Why it matters |
|----------|----------------|
| Use IAM Roles for EC2, Lambda, and other services | Roles rotate automatically; keys on a disk do not |
| Assign permissions through groups, not direct attachment | Auditable and consistent as the team grows |
| Apply least privilege from the start | Starting broad and narrowing later never happens |
| Require MFA for every human user | Blocks the entire class of leaked-password compromise |
| Avoid creating access keys unless unavoidable | Every key is a long-lived secret that must be rotated |
| Rotate credentials on a schedule and audit their age | Use the credential report; stale keys are a top finding |
| Audit with CloudTrail and IAM Access Advisor | Access Advisor shows which granted permissions are never used |
| Enable an account password policy | Enforces length, complexity, and reuse rules centrally |

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Attaching policies directly to users | Inconsistent permissions, unauditable drift | Move to groups, then re-attach at the group level |
| Giving a user `AdministratorAccess` for convenience | One compromised key is a full account breach | Grant only the required actions and resources |
| Creating access keys and forgetting them | Leaked keys used from unfamiliar IPs | Use the credential report; deactivate unused keys |
| Storing keys in code or a shared drive | Keys leak to anyone with repo or file access | Use roles, OIDC federation, or Secrets Manager |
| Password without `--password-reset-required` | The creator knows the user's password permanently | Always set the reset-required flag |
| Plain `Bool` on the MFA condition | User locked out and cannot enrol MFA | Use `BoolIfExists` so enrolment calls pass |
| Deleting a user that owns resources | Orphaned resources and failed automation | Detach policies, remove from groups, deactivate keys, then delete |
| No MFA on the root account | Root is the highest-value target in the account | Enable MFA on root and never create root access keys |

---

## Cost Considerations

IAM users, groups, policies, and MFA devices are all **free**. There is no charge for creating identities or for the API calls that manage them.

The cost dimension here is entirely about what a compromised user can spend. An access key with `AdministratorAccess` and no guardrails can launch GPU instances, create large EBS volumes, and transfer data out at a rate that reaches thousands of dollars within hours. That is why the mitigating controls — least privilege, MFA, budget alarms, and Service Control Policies — are cost controls as much as security controls.

Two practical cost habits:

- **Set a budget alarm before you start any lab.** A small threshold on the account catches an experiment you forgot about, and it catches a compromised key well before the end of the month.
- **Tag users with `Owner` and `Purpose`.** CloudTrail records the user ARN on every API call, so a tag makes it straightforward to attribute an unexpected charge to a person.

CloudTrail management events are free for the first trail, so the audit trail that would let you investigate a compromise costs nothing to have running.

---

## Security Considerations

- **Prefer federation over users.** IAM Identity Center gives humans temporary credentials through your existing identity provider, with MFA enforced centrally and no per-user passwords to manage.
- **Root account is not a user.** Enable MFA on root, never create root access keys, and do not use root for daily work.
- **Access keys are the main risk in this lesson.** They never expire, they work from anywhere, and `CreateAccessKey` is one of the highest-signal CloudTrail events. Alert on it.
- **Apply the MFA condition policy.** A JSON deny conditioned on `aws:MultiFactorAuthPresent` is far stronger than a policy that merely asks users to enrol.
- **Use IAM Access Analyzer.** It surfaces unused permissions and unintended external access, and it validates policies before you deploy them.
- **Watch for privilege escalation paths.** `iam:CreatePolicyVersion`, `iam:AttachUserPolicy`, and `iam:PutUserPolicy` let a user grant themselves more access. Restrict these permissions carefully.
- **Never share a user between humans.** Shared credentials destroy attribution — you can no longer tell who did what from CloudTrail, which is exactly what you need during an incident.

---

## Exercise

- Create a user `app-deployer` with programmatic access only.
- Attach `AmazonEC2FullAccess` and `AmazonS3FullAccess` policies — note that in a real account you would scope these far more tightly, and the point of the exercise is to then observe how broad they are.
- Generate access keys and configure the AWS CLI using:

```bash
aws configure --profile app-deployer
```

Then create an S3 bucket using only that profile. Afterwards, use IAM Access Advisor or the credential report to answer: which of the granted permissions did you actually use?

Success looks like: the CLI configuration is stored under the named profile `app-deployer`, `aws s3 mb` succeeds using only that profile's credentials, and you can articulate at least three of the granted permissions you never needed — which is the argument for least privilege stated in your own evidence.

---

## Resources

- [AWS IAM Users Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html)
- [IAM Policy Examples](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_examples.html)
- [Security Best Practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Credential Reports](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_getting-report.html)
- [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)
- [AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)

---

## Next Steps

This lesson covered how to create and configure IAM users, assign correct permissions via policies or groups, and apply AWS IAM best practices for user management. The next lesson covers the identity type you should reach for in almost every non-human scenario — and the one that removes long-lived credentials entirely.

> ➡️ Next: [IAM Roles](./02-iam-roles.md) · 🏠 [Phase overview](./README.md)
