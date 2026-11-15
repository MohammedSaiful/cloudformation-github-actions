# Automated AWS Infrastructure as Code with GitHub Actions and CloudFormation

A CI/CD pipeline that deploys AWS infrastructure automatically on every push to `main`. GitHub Actions authenticates to AWS using **OpenID Connect (OIDC)**, so no long-lived access keys are stored anywhere. It then validates and deploys a **CloudFormation** stack that provisions an **Ubuntu 24.04 LTS EC2 instance running Nginx**.


## Architecture

```text
Developer (VS Code)
      |
      | git add -> git commit -> git push origin main
      v
GitHub repository (cloudformation.yaml + .github/workflows/deploy.yml)
      |
      | push triggers workflow
      v
GitHub Actions runner (ubuntu-latest)
      |  1. Checkout repository
      |  2. Assume IAM role via OIDC (short-lived credentials)
      |  3. aws cloudformation validate-template
      |  4. aws cloudformation deploy
      v
AWS CloudFormation stack: bongodev-github-iac-dev
      |
      +--> VPC + public subnet + Internet gateway + route table
      +--> Security group (HTTP/80 only, no inbound SSH)
      +--> Ubuntu 24.04 EC2 (t3.micro)
                |
                | EC2 User Data: apt-get install nginx
                v
          Nginx web page: "Hello from bongoDev!"
```

## What Gets Provisioned

| AWS resource | Purpose |
| --- | --- |
| VPC (`10.42.0.0/16`) | Isolated network for the demo |
| Public subnet (`10.42.1.0/24`) | Hosts the EC2 instance |
| Internet gateway + route table | Internet access for installing and serving Nginx |
| Security group | Allows inbound TCP 80 only; no SSH |
| EC2 instance (`t3.micro`) | Ubuntu 24.04 LTS, latest AMI via Canonical's public SSM parameter |
| EC2 User Data | Installs and starts Nginx on first boot |
| Stack outputs | `WebsiteURL`, `InstanceId`, `PublicIPAddress`, `VpcId` |

## Repository Structure

```text
.
├── README.md
├── cloudformation.yaml                  # Ubuntu + Nginx + networking IaC
├── .github/
│   └── workflows/
│       └── deploy.yml                   # CI/CD workflow
├── iam/
│   ├── deployment-policy-lab.json       # Deployment permissions policy
│   ├── trust-policy-new-repos-2026.json # OIDC trust policy (new repos)
│   └── trust-policy-legacy-repos.json   # OIDC trust policy (older repos)
├── docs/
│   └── student-submission-checklist.md
└── .gitignore
```

## How the Pipeline Works

The workflow in `.github/workflows/deploy.yml` runs on:

```yaml
on:
  push:
    branches: [main]
    paths:
      - cloudformation.yaml
      - .github/workflows/deploy.yml
  workflow_dispatch:
```

It runs when `cloudformation.yaml` or the workflow file changes on `main`, and it can also be triggered manually. Changes to the README alone do not trigger a deployment.

**Pipeline steps:**

1. **Checkout repository** to get the template onto the runner.
2. **Configure temporary AWS credentials (OIDC)** by assuming the IAM role.
3. **Confirm AWS identity** to show the assumed-role caller identity.
4. **Validate CloudFormation template** with AWS.
5. **Create or update the stack** with:

   ```bash
   aws cloudformation deploy \
     --template-file cloudformation.yaml \
     --stack-name "$STACK_NAME" \
     --parameter-overrides Environment=dev \
     --region "$AWS_REGION" \
     --no-fail-on-empty-changeset
   ```
6. **Print outputs**, including the website URL.

`deploy` creates the stack on the first run and updates the same stack on later runs. `--no-fail-on-empty-changeset` treats a run with no changes as a success.

## Secure Authentication with OIDC

No AWS access keys are stored in GitHub. Instead:

1. GitHub issues a short-lived OIDC token to the workflow run.
2. AWS IAM checks the token against the role's **trust policy** (audience `sts.amazonaws.com` and the exact repository and branch subject).
3. If it matches, AWS issues temporary credentials that expire automatically.

The trust policy is locked to this repository's `main` branch. For repositories created on or after July 15, 2026, the subject includes the numeric owner and repository IDs:

```text
repo:<OWNER>@<OWNER_ID>/<REPO_NAME>@<REPO_ID>:ref:refs/heads/main
```

### Configuration used

| Setting | Value |
| --- | --- |
| AWS region | `us-west-2` |
| IAM role | `github-iac-deploy-role` |
| IAM policy | `github-iac-lab-policy` |
| CloudFormation stack | `github-iac-dev` |
| Environment parameter | `dev` |

GitHub repository **variables** (Settings > Secrets and variables > Actions > Variables):

| Name | Value |
| --- | --- |
| `AWS_ROLE_ARN` | ARN of the deployment role |
| `AWS_REGION` | `us-west-2` |

## Setup Summary

1. Create an empty GitHub repository.
2. Look up the GitHub owner ID and repository ID using the GitHub API.
3. Add the GitHub OIDC identity provider in AWS IAM (`https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`).
4. Create the IAM policy from `iam/deployment-policy-lab.json` and an IAM role with the completed trust policy.
5. Add the `AWS_ROLE_ARN` and `AWS_REGION` repository variables.
6. Push the project to `main`. The workflow runs and deploys the stack.

## Verifying the Deployment

1. In **GitHub > Actions**, confirm the workflow run is green.
2. In **AWS CloudFormation**, confirm the stack is `CREATE_COMPLETE`.
3. Open the `WebsiteURL` output in a browser. The page shows:

   ```text
   Hello from bongoDev!
   Infrastructure as Code is working!
   Ubuntu 24.04 LTS + Nginx + AWS CloudFormation
   Environment: dev
   ```

## Demonstrating an Update (Second Push)

To prove that a Git change updates the same infrastructure, the `Course` tag on the EC2 instance was changed from `DevOps` to `DevOps-Homework-V2` and pushed to `main`. The workflow ran again, the stack moved to `UPDATE_COMPLETE`, and the existing instance kept its ID with the new tag.

A tag change was used instead of editing the page HTML because User Data runs only on the first boot. Changing it in the template would not re-run the Nginx installation on an existing instance.

## Screenshots


```markdown
![GitHub Actions success](images/01_github_action_initial.jpg)
![GitHub Actions success](images/04_GitHub_Actions_second%20update.jpg)
![EC2 instance and tags](images/02_Ubuntu_EC2_instance.jpg)
![Nginx website](images/03_Nginx_website.jpg)
```

Make sure no access keys or sensitive account details are visible in any screenshot.

## Key Concepts

- **Infrastructure as Code (IaC):** infrastructure is described in a version-controlled template.
- **GitHub Actions:** runs the deployment automation.
- **CloudFormation:** reconciles AWS resources to match the template.
- **OIDC:** gives short-lived credentials, which is safer than long-lived access keys.

## Limitations

- This is a **training setup, not production-ready**. The site uses HTTP on port 80, and the IAM policy uses `Resource: "*"` for some EC2 and VPC actions for simplicity.
- A `CREATE_COMPLETE` stack does not guarantee that Nginx has finished installing, because User Data may still be running.
- The AMI is the latest Ubuntu 24.04 at deploy time, so builds are not strictly reproducible.

For production, restrict permissions by resource and tag, use CloudFormation service roles and approvals, protect the `main` branch, and pin AMI and package versions.

## Cleanup

Resources were deleted after the evidence was captured, to avoid AWS charges:

1. Delete the `github-iac-dev` stack in AWS CloudFormation.
2. Confirm the EC2 instance, VPC and networking resources are gone.
3. Disable the GitHub Actions workflow so a later push does not recreate the stack.
4. Delete the lab IAM role and policy when no longer needed.

## Security

- No AWS credentials are stored in this repository or in GitHub Secrets.
- Do not commit access keys, private keys or other secrets.
- The IAM trust policy is restricted to a single repository and branch. Never use wildcards for the repository or branch.


# Question Answers: GitHub Actions + CloudFormation (OIDC)

*Automate AWS Infrastructure as Code with GitHub Actions*.

> **Definition to remember:** IaC = describe the desired infrastructure as code; GitHub Actions = run the deployment automation; CloudFormation = reconcile AWS resources to that template.

---

## 1. What is the difference between GitHub Actions and AWS CloudFormation?

**GitHub Actions** is a CI/CD automation service. It runs workflow steps (checkout, validate, deploy) on a runner when an event such as a push happens. It does not know which AWS resources you want.

**AWS CloudFormation** is AWS's infrastructure-as-code service. It reads a template describing the desired resources (VPC, EC2, security group) and creates, updates or deletes them to match.

| | GitHub Actions | AWS CloudFormation |
| --- | --- | --- |
| Role | Runs the automation | Provisions the resources |
| Input | Workflow YAML (`deploy.yml`) | Template YAML (`cloudformation.yaml`) |
| Trigger | Git events (push, manual run) | API or CLI calls |

In this lab, Actions calls `aws cloudformation deploy`, and CloudFormation does the actual work.

---

## 2. Why can a `git push` deploy infrastructure without anyone logging into the AWS console?

- The push triggers the workflow because the changed file matches the `paths` filter.
- The runner authenticates to AWS on its own by assuming an IAM role through OIDC.
- The runner runs the CloudFormation deploy command.
- The infrastructure is already described in `cloudformation.yaml`, so no manual clicking is needed.

Git becomes the source of truth, and the pipeline does what a person would otherwise do in the console.

---

## 3. What does OIDC do, and why is it safer than committing AWS access keys?

OpenID Connect lets GitHub issue a signed, short-lived identity token to each workflow run. AWS checks that token against the IAM role's trust policy (audience `sts.amazonaws.com` and the exact repository and `main` branch). If it matches, AWS issues temporary credentials that expire automatically.

**Why it is safer than access keys:**

- No permanent secret exists, so nothing sensitive is stored in the repo or in GitHub Secrets.
- Credentials expire quickly, so a stolen one is only briefly useful.
- Access is limited to one repository and branch, while an access key works from anywhere.
- There are no keys to rotate.

---

## 4. On a second push, why is the existing stack updated instead of creating a new one?

`aws cloudformation deploy` looks for a stack with the given name (`bongodev-github-iac-dev`):

- If it does not exist, it creates it.
- If it exists, it creates a change set that compares the new template with the current one and applies only the differences.

Because the stack name is the same, the second push updates the same stack. A tag-only change updates the instance in place, so the instance ID stays the same.

---

## 5. Is CloudFormation stack creation complete always the same as Nginx ready? Why not?

**No.**

- CloudFormation reports `CREATE_COMPLETE` once AWS has created all the resources. For the instance, this means it is launched and running.
- Nginx is installed by the EC2 **User Data** script, which runs after boot as a separate process.
- CloudFormation does not wait for it unless you add a signal such as `cfn-signal` with a `CreationPolicy`.

So the stack can be complete while `apt-get` is still installing Nginx, and the website may not respond for a short time. If the script fails, the stack still shows success.

---

## 6. Why does this homework update the EC2 tag instead of changing the webpage HTML?

- The HTML is written by the User Data script, which by default runs only on the instance's **first boot**.
- Changing User Data in the template does not re-run the script on an existing instance. It may even cause the instance to be replaced.
- A tag change is a safe in-place update that keeps the same instance ID. This clearly shows that a Git push updates existing infrastructure.

For real application changes, use an application deployment pipeline, SSM automation, or instance replacement.

---

## 7. What should happen to AWS resources when you have finished the assignment?

Delete them to avoid charges, after capturing screenshots and evidence:

1. Delete the `bongodev-github-iac-dev` stack in CloudFormation. This removes the EC2 instance, VPC, subnet, internet gateway, route table and security group.
2. Confirm the stack shows `DELETE_COMPLETE`, the instance is terminated, and no network resources remain. Investigate any `DELETE_FAILED` errors.
3. Disable the GitHub Actions workflow so a later push does not recreate the stack.
4. Delete the lab IAM role and policy when no longer needed. Keep the GitHub OIDC provider if another project uses it.

Do not push the deployment files again after cleanup, or the workflow may recreate everything.