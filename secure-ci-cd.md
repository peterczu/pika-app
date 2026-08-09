# AWS IAM & GitHub Actions OIDC Policy Commands

## Purpose

This document records the AWS CLI commands used to create and secure a dedicated GitHub Actions deployment role for the Pika application.

The goal is to follow the principle of **least privilege** instead of allowing the application deployment workflow to use `AdministratorAccess`.

---

## 1. Inspect an IAM Role

### Get role information

```bash
aws iam get-role \
  --role-name github-actions-terraform-role \
  --query 'Role.[RoleName,Arn]' \
  --output table
```

### Inspect the OIDC trust policy

```bash
aws iam get-role \
  --role-name github-actions-terraform-role \
  --query 'Role.AssumeRolePolicyDocument' \
  --output json
```

This shows **who is allowed to assume the role** through GitHub's OIDC provider.

---

## 2. Inspect Role Permissions

### List attached managed policies

```bash
aws iam list-attached-role-policies \
  --role-name github-actions-terraform-role \
  --output table

### List inline policies

```bash
aws iam list-role-policies \
  --role-name github-actions-terraform-role \
  --output table
```

These commands help determine what permissions a role currently has.

---

## 3. Create a GitHub Actions OIDC Trust Policy

Example trust policy file:

```text
pika-app-trust-policy.json
```

The important security condition is:

```json
"token.actions.githubusercontent.com:sub": "repo:peterczu@116700365/pika-app@1328129767:ref:refs/heads/main"
```

This restricts the role to the specific:

* GitHub owner
* repository
* repository ID
* `main` branch

The audience must also be:

```json
"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
```

---

## 4. Validate JSON Policy Documents

Before sending a JSON policy to AWS:

```bash
python3 -m json.tool pika-app-trust-policy.json
```

For the deployment policy:

```bash
python3 -m json.tool pika-app-deployment-policy.json
```

A clean result means the JSON is syntactically valid.

### Common error

If you see:

```text
Extra data: line 41 column 1
```

the file usually contains **more than one JSON object**.

The policy must contain one complete JSON document.

---

# 5. Create the Dedicated Application Deployment Role

```bash
aws iam create-role \
  --role-name github-actions-pika-app-role \
  --assume-role-policy-document file://pika-app-trust-policy.json \
  --description "GitHub Actions deployment role for pika-app"
```

This creates the role but does **not** automatically give it AWS permissions.

This creates the role but does **not** automatically give it AWS permissions.

---

## 6. Verify the New Role

```bash
aws iam get-role \
  --role-name github-actions-pika-app-role \
  --query 'Role.[RoleName,Arn]' \
  --output table
```

Get only the ARN:

```bash
aws iam get-role \
  --role-name github-actions-pika-app-role \
  --query 'Role.Arn' \
  --output text
```

Expected format:

```text
arn:aws:iam::ACCOUNT_ID:role/github-actions-pika-app-role
```

---

# 7. Create a Least-Privilege Deployment Policy

Example policy:

```text
pika-app-deployment-policy.json
```

The policy contains permissions for:

```text
ECR
 ├── Authenticate
 ├── Upload Docker layers
 └── Push image

ECS
 ├── Describe task definition
 ├── Register task definition
 ├── Update service
 └── Describe service

IAM
 └── Pass only the ECS execution role
```

The important security principle is:

```text
Do not use iam:*
Do not use AdministratorAccess
```

Instead:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::ACCOUNT_ID:role/pika-ecs-execution-role"
}
```

---

# 8. Create the Managed Policy

```bash
aws iam create-policy \
  --policy-name github-actions-pika-app-deployment \
  --policy-document file://pika-app-deployment-policy.json \
  --description "Least-privilege ECR and ECS deployment permissions for pika-app"
```

This creates a **customer-managed IAM policy**.

---

# 9. Attach the Policy to the Role

```bash
aws iam attach-role-policy \
  --role-name github-actions-pika-app-role \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/github-actions-pika-app-deployment
```

---

# 10. Verify Attached Policies

```bash
aws iam list-attached-role-policies \
  --role-name github-actions-pika-app-role \
  --output table
```

The application role should show:

```text
github-actions-pika-app-deployment
```

It should **not** have:

```text
AdministratorAccess
```
# 11. Update an Existing Customer-Managed Policy

Once a managed policy exists, don't run `create-policy` again just because the JSON changed.

Create a **new policy version** instead.

```bash
aws iam create-policy-version \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/github-actions-pika-app-deployment \
  --policy-document file://pika-app-deployment-policy.json \
  --set-as-default
```

The:

```text
--set-as-default
```

option makes the new policy version active.

---

# 12. Find the Current Default Policy Version

```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/github-actions-pika-app-deployment \
  --query 'Policy.DefaultVersionId' \
  --output text
```

Example:

```text
v2
```

---

# 13. Inspect a Specific Policy Version

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/github-actions-pika-app-deployment \
  --version-id v2 \
  --output json
```

To specifically inspect the `PassRole` statement:

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/github-actions-pika-app-deployment \
  --version-id v2 \
  --query 'PolicyVersion.Document.Statement[?Sid==`PassECSTaskExecutionRole`]' \
  --output json
```

---

# 14. Why `iam:PassRole` Was Required

The GitHub Actions workflow registers an ECS task definition.

The task definition specifies:

```text
pika-ecs-execution-role
```

AWS therefore requires the GitHub Actions caller to have permission to pass that role to ECS.

Without it, deployment fails with:

```text
is not authorized to perform:
iam:PassRole
```

The correct solution is **not** to give GitHub general IAM access.

Instead:

```text
GitHub Actions
      │
      ▼
github-actions-pika-app-role
      │
      └── iam:PassRole
              │
              ▼
      pika-ecs-execution-role
```

This limits exactly which role GitHub can pass.

---

# 15. GitHub Actions Configuration

The application workflow should assume the dedicated role:

```yaml
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: ${{ secrets.AWS_PIKA_APP_ROLE_ARN }}
    aws-region: eu-north-1
```

The ARN is stored in GitHub Actions Secrets rather than hard-coded into the workflow.

---

# 16. Security Architecture

### Before

GitHub Actions
      │
      ▼
github-actions-terraform-role
      │
      ▼
AdministratorAccess
      │
      ▼
Entire AWS Account
```

This works, but has a large blast radius.

### After

```text
GitHub Actions
      │
      ▼
OIDC
      │
      ▼
github-actions-pika-app-role
      │
      ├── ECR
      │
      ├── ECS
      │
      └── iam:PassRole
              │
              ▼
      pika-ecs-execution-role
```

Terraform infrastructure workflows and application deployment workflows can therefore have separate permissions.

---
# 17. Important Lesson

IAM troubleshooting should not immediately lead to broader permissions.

When AWS reports:

```text
not authorized to perform iam:PassRole
```

the correct approach is:

```text
1. Read the denied action
2. Identify why the action is required
3. Identify the exact resource
4. Grant only that action
5. Retry
6. Repeat if another legitimate permission is missing
```

This is the practical application of **least privilege**.

---

# 18. Useful IAM Investigation Sequence

For future projects:

```bash
# Inspect role
aws iam get-role --role-name ROLE_NAME

# List managed policies
aws iam list-attached-role-policies \
  --role-name ROLE_NAME

# List inline policies
aws iam list-role-policies \
  --role-name ROLE_NAME

# Inspect policy
aws iam get-policy \
  --policy-arn POLICY_ARN

# Find default policy version
aws iam get-policy \
  --policy-arn POLICY_ARN \
  --query 'Policy.DefaultVersionId'

# Inspect policy version
aws iam get-policy-version \
  --policy-arn POLICY_ARN \
  --version-id VERSION_ID
```

---

## Pika Lesson 4 Milestone

At this point the Pika project has demonstrated:

```text
Infrastructure as Code
        ↓
Terraform
        ↓
ECS/Fargate + ALB
        ↓
ECR
        ↓
GitHub OIDC
        ↓
Immutable Git SHA images
        ↓
Automated ECS deployment
        ↓
Dedicated deployment IAM role
        ↓
Least-privilege ECR/ECS permissions
        ↓
Explicit iam:PassRole

This is a significant step beyond simply making a GitHub Actions workflow deploy successfully.
