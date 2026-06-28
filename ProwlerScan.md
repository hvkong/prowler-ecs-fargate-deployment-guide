# ProwlerScan Role Setup Guide

This guide explains how to create the ProwlerScan IAM role in the AWS account(s) you
want Prowler to scan. This role is assumed by the Prowler Worker container running on ECS
Fargate.

## Overview

The ProwlerScan role is deployed in the **target account** (the account being scanned).
It grants Prowler read-only access to audit your AWS resources for security findings.

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│  Prowler Account (ECS)      │         │  Target Account             │
│                             │         │                             │
│  Worker Container           │         │  ProwlerScan Role           │
│    └─ Task Role             │────────►│    ├─ SecurityAudit          │
│       (sts:AssumeRole)      │  STS    │    ├─ ViewOnlyAccess         │
│                             │         │    ├─ ProwlerAdditions       │
│                             │         │    └─ Trust Policy            │
│                             │         │       (ExternalId required)   │
└─────────────────────────────┘         └─────────────────────────────┘
```

## Prerequisites

Before deploying this template, you need:

1. **Prowler Account ID** — the 12-digit AWS account ID where your Prowler ECS stack runs
2. **External ID** — the tenant UUID shown in the Prowler UI

### Getting the External ID

The External ID is your Prowler tenant UUID. It is auto-generated when you first sign up
and cannot be changed. To find it:

1. Log into Prowler at `https://<your-domain>`
2. Navigate to **Providers → Add Provider → AWS**
3. Select **IAM Role** as the credential type
4. The **External ID** field is pre-filled and disabled — this is your tenant UUID
5. Copy this value (use the copy button next to it)

This same External ID is used for all providers and integrations in your Prowler tenant.

## Deployment

### Option A: CloudFormation (Console)

1. Open the [CloudFormation console](https://console.aws.amazon.com/cloudformation) in the **target account**
2. Click **Create stack → With new resources**
3. Upload `prowler-scan-role.yaml`
4. Fill in the parameters:
   - **ProwlerAccountId**: Your Prowler ECS account ID
   - **ExternalId**: The tenant UUID from Prowler UI
   - **EnableAdditionsPolicy**: `true` (recommended for full scan coverage)
   - **EnableS3Integration**: `true` if you plan to use the S3 export feature
   - **EnableSecurityHubIntegration**: `true` if you plan to send findings to Security Hub
5. Acknowledge IAM capability and create the stack
6. Copy the `ProwlerScanRoleArn` from the Outputs tab

### Option B: CloudFormation (CLI)

```bash
aws cloudformation create-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --stack-name ProwlerScan \
  --template-body file://prowler-scan-role.yaml \
  --parameters \
    ParameterKey=ProwlerAccountId,ParameterValue=<YOUR_PROWLER_ACCOUNT_ID> \
    ParameterKey=ExternalId,ParameterValue=<YOUR_EXTERNAL_ID>
```

With S3 integration enabled:

```bash
aws cloudformation create-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --stack-name ProwlerScan \
  --template-body file://prowler-scan-role.yaml \
  --parameters \
    ParameterKey=ProwlerAccountId,ParameterValue=<YOUR_PROWLER_ACCOUNT_ID> \
    ParameterKey=ExternalId,ParameterValue=<YOUR_EXTERNAL_ID> \
    ParameterKey=EnableS3Integration,ParameterValue=true \
    ParameterKey=S3BucketName,ParameterValue=<YOUR_BUCKET_NAME> \
    ParameterKey=S3BucketAccountId,ParameterValue=<BUCKET_OWNER_ACCOUNT_ID>
```

## Connecting the Provider in Prowler

After deploying the role:

1. Copy the Role ARN from the CloudFormation Outputs (format: `arn:aws:iam::<target-account>:role/ProwlerScan`)
2. In Prowler UI, go to **Providers → Add Provider**
3. Select **AWS** as the provider type
4. Choose **IAM Role** as the credential type
5. Paste the Role ARN
6. The External ID is already filled in (your tenant UUID)
7. Click **Connect** — Prowler will test the connection by calling `sts:AssumeRole`

If the connection test passes, you can launch a scan against this account.

## What Permissions Are Granted

| Policy | Source | Purpose |
|--------|--------|---------|
| `SecurityAudit` | AWS managed policy | Core read-only security auditing |
| `ViewOnlyAccess` | AWS managed policy | Read-only access to AWS resource configurations |
| `ProwlerAdditions` | Inline (optional) | Additional read-only permissions for services not covered by the managed policies (Bedrock, Backup, AppStream, API Gateway, etc.) |
| `ProwlerS3Integration` | Inline (optional) | `s3:PutObject`, `s3:ListBucket`, `s3:DeleteObject` on the specified bucket for report export |
| `ProwlerSecurityHub` | Inline (optional) | `securityhub:BatchImportFindings`, `securityhub:BatchUpdateFindings`, `securityhub:GetFindings` for sending findings to Security Hub |

### About the Additions Policy

The `SecurityAudit` and `ViewOnlyAccess` managed policies cover most AWS services, but
some checks require permissions not included in those policies. The `ProwlerAdditions`
inline policy (enabled by default) adds read-only permissions for:

- AWS Bedrock, Backup, AppStream, Directory Service
- CodeBuild, CodeArtifact, DLM, DRS
- API Gateway (GET on REST APIs and HTTP APIs)
- Security Hub (finding import for the integration feature)
- Shield, WAF, Macie, Lightsail
- Various other services with limited coverage in managed policies

This policy is based on Prowler's official
[`prowler-additions-policy.json`](https://github.com/prowler-cloud/prowler/blob/master/permissions/prowler-additions-policy.json).
Disabling it will cause some checks to return errors or skip findings for those services.

## Security Model

**External ID prevents confused deputy attacks.** The trust policy requires the caller
to present the correct External ID (your Prowler tenant UUID) when assuming the role. This
ensures that even if another service in the same Prowler account has `sts:AssumeRole`
permission, it cannot assume this role without knowing the External ID.

**Scoped trust.** The trust policy restricts assumption to the specific Prowler account
(`ProwlerAccountId`). No other AWS account can assume this role.

**Read-only access.** All policies grant read-only permissions. The only write operations
are optional: S3 PutObject (for report export) and Security Hub BatchImportFindings (for
finding export). No destructive or modify operations are included.

## Updating the Role

To enable features after initial deployment (e.g., adding S3 integration later):

```bash
aws cloudformation update-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --stack-name ProwlerScan \
  --template-body file://prowler-scan-role.yaml \
  --parameters \
    ParameterKey=ProwlerAccountId,UsePreviousValue=true \
    ParameterKey=ExternalId,UsePreviousValue=true \
    ParameterKey=EnableS3Integration,ParameterValue=true \
    ParameterKey=S3BucketName,ParameterValue=<YOUR_BUCKET_NAME> \
    ParameterKey=S3BucketAccountId,ParameterValue=<BUCKET_OWNER_ACCOUNT_ID>
```

## Multi-Account Scanning

To scan multiple AWS accounts, deploy this template in each account with the same
`ProwlerAccountId` and `ExternalId` values. Each deployment creates a `ProwlerScan` role
that trusts your Prowler ECS account.

Then add each account as a separate provider in Prowler UI with the respective Role ARN.

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Connection test fails with "Access Denied" | External ID mismatch | Verify the External ID in Prowler UI matches the one in the CloudFormation parameter |
| Connection test fails with "is not authorized to perform: sts:AssumeRole" | Task role missing permission | Ensure the Prowler ECS task role has `sts:AssumeRole` on this role's ARN (see main README Step 5) |
| Scan shows many "access denied" errors | Additions policy disabled | Enable `EnableAdditionsPolicy` via stack update |
| S3 integration connection test fails | S3 permissions not enabled | Update stack with `EnableS3Integration=true` and correct bucket parameters |
| Security Hub findings not appearing | Integration not enabled or Security Hub not set up | Enable `EnableSecurityHubIntegration`, ensure Security Hub is enabled in the account, and accept the Prowler partner integration |

## Related Documentation

- [Main ECS Deployment Guide](README.md) — Step 5 covers the ECS task role that calls `sts:AssumeRole`
- [Prowler AWS Authentication](https://docs.prowler.com/user-guide/providers/aws/authentication) — Official Prowler documentation
- [Prowler S3 Integration](https://docs.prowler.com/user-guide/tutorials/prowler-app-s3-integration) — Official S3 integration guide
- [Prowler Security Hub Integration](https://docs.prowler.com/user-guide/tutorials/prowler-app-security-hub-integration) — Official Security Hub integration guide
- [prowler-additions-policy.json](https://github.com/prowler-cloud/prowler/blob/master/permissions/prowler-additions-policy.json) — Source of the additions policy permissions
