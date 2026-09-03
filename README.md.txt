# AWS IAM Least-Privilege & MFA Governance Project

This repository contains the architecture, policies, and documentation for a multi-layered AWS security strategy. It establishes user groups with scoped least-privilege policies, introduces a global multi-factor authentication (MFA) enforcement framework, and outlines an organizational governance structure using Service Control Policies (SCPs).

---

## 1. IAM Groups and Least-Privilege Setup

The account structure partitions users into dedicated groups to maintain separation of duty and prevent privilege creep.

### ReadOnly Group
* **Policy Attached:** `ReadOnlyAccess` (AWS Managed Policy)
* **Description:** Grants read-only access to all AWS resources across the account.
* **Test User:** `readOnly`

### Developers Group
* **Policy Attached:** `Developers-LeastPrivilege` (Customer Managed/Inline Policy)
* **Description:** Grants limited access strictly to Amazon S3 development assets, ensuring developers cannot modify global infrastructure or change administrative parameters.
* **Test User:** Associated developer test identity.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DevS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 2. Multi-Factor Authentication (MFA) Enforcement Policy

To meet compliance and protect against credential compromise, a strict MFA gatekeeper policy is assigned to all groups. This policy explicitly **denies all actions** across the entire AWS account unless the user has successfully authenticated using a verified MFA device. 

The policy permits unauthenticated users to manage only their own passwords and MFA configurations to prevent self-lockout during initial setup.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowViewAccountInfoAndMFA",
      "Effect": "Allow",
      "Action": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ResyncMFADevice",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:GetUser",
        "iam:ChangePassword"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyAllExceptMFASetupUnlessMFAIsPresent",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ResyncMFADevice",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:GetUser",
        "iam:ChangePassword"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

---

## 3. AWS Organizations Architecture (CloudTech Solutions)

The diagram below outlines the logical Multi-Account framework for a fictional enterprise entity, **CloudTech Solutions**. It separates administrative, operational, and development sandbox concerns into isolated Organizational Units (OUs).

```mermaid
graph TD
    classDef mgmt fill:#FF9900,stroke:#333,stroke-width:2px,color:#fff;
    classDef ou fill:#1A73E8,stroke:#333,stroke-width:2px,color:#fff;
    classDef acct fill:#34A853,stroke:#333,stroke-width:2px,color:#fff;

    Root[CloudTech Solutions Root] --> Management[Management Account]
    Root --> CoreOU[Core Services OU]
    Root --> WorkloadsOU[Workloads OU]
    Root --> SandboxOU[Sandbox OU]

    CoreOU --> LogAcct[Log Archive Account]
    CoreOU --> SecAcct[Security Tooling Account]

    WorkloadsOU --> DevAcct[Development Account]
    WorkloadsOU --> ProdAcct[Production Account]

    SandboxOU --> SandboxAcct[Developer Sandbox Account]

    class Management mgmt;
    class CoreOU,WorkloadsOU,SandboxOU ou;
    class LogAcct,SecAcct,DevAcct,ProdAcct,SandboxAcct acct;
```

---

## 4. Organizational Governance via Service Control Policy (SCP)

This standard production guardrail policy is designed to be applied at the Organization Root level. It acts as an immutable perimeter that ensures member accounts cannot decouple themselves from central compliance mechanisms or sabotage infrastructure tracking.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectOrganizationMembership",
      "Effect": "Deny",
      "Action": [
        "organizations:LeaveOrganization"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PreventCloudTrailDisabling",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 5. Security Control Comparison Matrix: SCP vs. IAM

| Architectural Feature | Service Control Policies (SCPs) | IAM Policies |
| :--- | :--- | :--- |
| **Deployment Level** | Attached to Organization Roots, OUs, or Member Accounts. | Attached directly to Users, User Groups, or execution Roles. |
| **Strategic Intent** | Functions as an absolute permission filter / max boundary guardrail. | Functionally grants operational capability to users or resources. |
| **Capability Granting** | **Cannot grant access.** Only permits or filters existing allowances. | **Can explicitly grant access** or deny operational actions. |
| **Hierarchy Precedence** | An explicit `Deny` in an SCP supersedes any local administrator `Allow`. | Subject to constraints and ceilings established by active root SCPs. |
