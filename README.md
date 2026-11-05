# bongoDev Homework: Infrastructure as Code with GitHub Actions and AWS CloudFormation

This project deploys an **Ubuntu 24.04 LTS EC2 instance running Nginx** inside a **new VPC** on AWS. Deployment is fully automated. A `git push` to `main` starts a GitHub Actions workflow, which signs in to AWS with **OpenID Connect (OIDC)** and creates or updates a CloudFormation stack. No AWS access keys are stored anywhere.

## Architecture

```text
Developer (git push to main)
        |
        v
GitHub repository (cloudformation.yaml + .github/workflows/deploy.yml)
        |
        v
GitHub Actions runner
   - checkout code
   - assume IAM role via OIDC (short-lived credentials)
   - validate CloudFormation template
   - deploy CloudFormation stack
        |
        v
AWS CloudFormation stack: bongodev-github-iac-dev (and -test)
   - VPC, public subnet, internet gateway, route table
   - Security group (HTTP port 80 only, no SSH)
   - Ubuntu 24.04 EC2 (t3.micro) with Nginx installed through User Data
```

## Repository structure

```text
.
├── README.md
├── cloudformation.yaml                  # VPC, subnet, routing, security group, EC2 + Nginx
├── iam-policy/
│   ├── deployment-policy-lab.json       # permissions policy for the deploy role
│   └── trust-policy.json                # OIDC trust policy (placeholders only)
├── screenshot-cloudformation/           # evidence screenshots
└── .github/
    └── workflows/
        └── deploy.yml                   # GitHub Actions deployment pipeline
```

## What was created

| Resource | Details |
|---|---|
| VPC | `10.42.0.0/16` |
| Public subnet | `10.42.1.0/24` |
| Internet gateway and route table | Internet access for the instance |
| Security group | Inbound TCP 80 from anywhere, no SSH |
| EC2 instance | Ubuntu 24.04 LTS, `t3.micro`, Nginx installed with `apt-get` |
| Stack outputs | `WebsiteURL`, `InstanceId`, `PublicIPAddress`, `VpcId` |

## Configuration used

| Setting | Value |
|---|---|
| GitHub repository | `bongodev-cloudformation-github-actions` |
| Branch | `main` |
| AWS region | `us-east-1` |
| IAM role | `bongodev-github-iac-policy-deploy-role` |
| IAM permissions policy | `bongodev-github-iac-lab-policy` |
| CloudFormation stacks | `bongodev-github-iac-dev`, `bongodev-github-iac-test` |
| GitHub repository variables | `AWS_ROLE_ARN`, `AWS_REGION` |

## IAM policies

The `iam-policy/` folder holds copies of the policies used in this lab. They are documentation only. Pushing them does not change anything in AWS.

- `iam-policy/deployment-policy-lab.json` is the permissions policy attached to the deploy role. It allows only the CloudFormation, SSM parameter and EC2/VPC actions needed for this lab.
- `iam-policy/trust-policy.json` is the trust policy that lets GitHub Actions assume the role through OIDC. It contains placeholders for the account ID and GitHub owner. Real values are not committed.

## How it works

1. **OIDC identity provider.** AWS IAM trusts `token.actions.githubusercontent.com` with the audience `sts.amazonaws.com`.
2. **IAM role with a trust policy.** The role can be assumed only by this repository on the `main` branch. GitHub repositories created after July 15, 2026 use the immutable subject format, which includes the numeric owner ID and repository ID:

   ```text
   repo:<GITHUB_OWNER>@<OWNER_ID>/<REPO_NAME>@<REPO_ID>:ref:refs/heads/main
   ```

3. **Permissions policy.** Limited to the actions in `iam-policy/deployment-policy-lab.json`.
4. **Workflow.** On a push to `main` that changes `cloudformation.yaml` or the workflow file, the pipeline runs these steps: checkout, configure AWS credentials via OIDC, confirm identity, validate the template, then `aws cloudformation deploy`.
5. **Create or update.** `aws cloudformation deploy` creates the stack on the first run and updates the same stack on later runs.


## Initial setup

Complete these steps once, in this order, before the first push.

| Step | Where | Action | Result |
|---|---|---|---|
| 1 | GitHub | Create an empty repository named `bongodev-cloudformation-github-actions` | Repository ready for the code |
| 2 | Browser | Open `https://api.github.com/repos/<GITHUB_OWNER>/bongodev-cloudformation-github-actions` and note the owner ID and repository ID | Numeric IDs for the trust policy |
| 3 | AWS IAM, Identity providers | Add an OpenID Connect provider with URL `https://token.actions.githubusercontent.com` and audience `sts.amazonaws.com` | AWS trusts GitHub tokens |
| 4 | AWS IAM, Policies | Create the policy `bongodev-github-iac-lab-policy` from `iam-policy/deployment-policy-lab.json` | Permissions for CloudFormation, SSM and EC2/VPC |
| 5 | AWS IAM, Roles | Create the deploy role with a custom trust policy based on `iam-policy/trust-policy.json`, attach the permissions policy | Role that only this repository and branch can assume |
| 6 | AWS IAM, Roles | Copy the role ARN from the role summary page | Value for the GitHub variable |
| 7 | GitHub, Settings, Secrets and variables, Actions, Variables | Add `AWS_ROLE_ARN` (the role ARN) and `AWS_REGION` (`us-east-1`) | Workflow can find the role and region |
| 8 | Local terminal | Set `git config user.name` and `user.email`, then add `cloudformation.yaml` and `.github/workflows/deploy.yml` | Project files ready |
| 9 | Local terminal | Run `git init`, `git branch -M main`, `git add .`, `git commit`, `git remote add origin <repo URL>` | Local repository linked to GitHub |
| 10 | Local terminal | Run `git push -u origin main` | Push starts the workflow |
| 11 | GitHub, Actions | Open the run and wait for the green check | Stack created by the pipeline |
| 12 | AWS CloudFormation (`us-east-1`) | Confirm `CREATE_COMPLETE` on `bongodev-github-iac-dev` and read the `WebsiteURL` output | Infrastructure verified |
| 13 | Browser | Open the `WebsiteURL` (HTTP) | Page shows "Hello from bongoDev!" and the environment |

Notes:

- The OIDC provider is created once per AWS account. If it already exists, reuse it.
- Replace `<GITHUB_OWNER>`, the account ID, and the numeric IDs with real values only in AWS. Do not commit them.
- Pushes that change only the README or the folders for policies and screenshots do not trigger the workflow.
## Deploying

**Automatic (dev):** push a change to `cloudformation.yaml` or `.github/workflows/deploy.yml` on `main`.

**Manual (dev or test):** open **GitHub → Actions → Deploy Ubuntu Nginx with CloudFormation → Run workflow**, choose the environment, and run it. The `test` environment creates a separate stack, `bongodev-github-iac-test`, with its own VPC and EC2 instance.

After the run succeeds, open the `WebsiteURL` shown in the **Print outputs** step. It starts with `http://`.

## Demonstrating an update

The EC2 tag in `cloudformation.yaml` was changed from `Course: bongoDev-DevOps` to `Course: bongoDev-DevOps-Homework-V2` and pushed to `main`. The workflow ran again, CloudFormation moved the same stack to `UPDATE_COMPLETE`, and the instance kept the same Instance ID with the new tag value.

A tag was used for the test, and not the web page HTML, because EC2 User Data runs only on the first boot. Changing it in the template does not re-run the install on an existing instance.


## What cloudformation.yaml creates

The template builds a complete, isolated web server environment from scratch. Nothing is created by hand in the console.

### Parameters

| Parameter | Type | Default | Purpose |
|---|---|---|---|
| `Environment` | String (`dev`, `test`, `prod`) | `dev` | Shown on the website and used in resource tags and names |
| `LatestUbuntuAmiId` | SSM parameter | Canonical Ubuntu 24.04 LTS path | Looks up the current official Ubuntu 24.04 AMI, so no region-specific AMI ID is needed |

### Resources

| Logical ID | AWS resource | Details | Purpose |
|---|---|---|---|
| `DemoVPC` | VPC | CIDR `10.42.0.0/16`, DNS support and hostnames enabled | Private network for the demo |
| `InternetGateway` | Internet gateway | Tagged with the stack name | Connects the VPC to the internet |
| `GatewayAttachment` | VPC gateway attachment | Attaches the gateway to the VPC | Enables internet access |
| `PublicSubnet` | Subnet | CIDR `10.42.1.0/24`, public IP on launch | Place where the instance runs |
| `PublicRouteTable` | Route table | Belongs to the VPC | Holds the routing rules |
| `DefaultRoute` | Route | `0.0.0.0/0` through the internet gateway | Sends outbound traffic to the internet |
| `PublicRouteAssociation` | Route table association | Links the subnet to the route table | Makes the subnet public |
| `WebSecurityGroup` | Security group | Inbound TCP 80 from `0.0.0.0/0`, no SSH | Allows web traffic only |
| `UbuntuNginxServer` | EC2 instance | Ubuntu 24.04 LTS, `t3.micro`, IMDSv2 required, tags `Name`, `Environment`, `Course` | Web server |

### What User Data does on the instance

On first boot the instance runs a script that:

1. Updates the package list and installs Nginx with `apt-get`.
2. Writes the demo web page to `/var/www/html/index.html`, including the `Environment` value.
3. Tests the Nginx configuration with `nginx -t`.
4. Enables and starts the Nginx service.
5. Writes a log to `/var/log/bongodev-bootstrap.log`.

User Data runs only on the first boot. Changing it later does not re-run it on an existing instance.

### Outputs

| Output | Value | Use |
|---|---|---|
| `WebsiteURL` | `http://<public IP>` | Address to open in the browser |
| `InstanceId` | EC2 instance ID | Identifies the server |
| `PublicIPAddress` | Public IPv4 address | Server address |
| `VpcId` | VPC ID | Identifies the new network |

## Cleanup

Both stacks were deleted from the AWS CloudFormation console (region `us-east-1`) after the evidence was captured, so no resources keep running. The deployment role has no delete permission by design. Stacks are deleted with a normal console user.


## Questions

### 1. What is the difference between GitHub Actions and AWS CloudFormation?

GitHub Actions is an automation service. It runs the pipeline steps (checkout, sign in to AWS, validate, deploy) when an event such as a push happens. CloudFormation is the AWS service that reads the YAML template and creates, updates or deletes the AWS resources to match it. GitHub Actions decides when and how the deployment runs, and CloudFormation does the actual provisioning.

### 2. Why can a git push deploy infrastructure without anyone logging into the AWS console?

The push triggers the workflow, and the workflow signs in to AWS on its own using a trusted IAM role. It then calls `aws cloudformation deploy` with the template. Everything happens through the AWS API, so nobody has to open the console or click Create stack.

### 3. What does OIDC do, and why is it safer than committing AWS access keys?

OIDC lets GitHub prove the identity of the workflow (this repository and this branch) with a signed token. AWS checks the token against the role's trust policy and, if it matches, issues temporary credentials that expire after about an hour. There is no permanent secret to store, leak, rotate or commit by mistake, and only this repository and branch can obtain credentials. Access keys are long-lived, and anyone who finds them can use them until they are revoked.

### 4. On a second push, why is the existing stack updated instead of creating a new one?

The workflow always uses the same stack name, `bongodev-github-iac-dev`. `aws cloudformation deploy` creates the stack if that name does not exist, and otherwise creates a change set and updates the existing stack. CloudFormation compares the new template with the current one and changes only what differs.

### 5. Is CloudFormation stack creation complete always the same as Nginx ready?

No. `CREATE_COMPLETE` only means AWS created the resources, including the EC2 instance. Nginx is installed afterward by the User Data script, which runs inside the instance on its first boot. CloudFormation does not wait for that script, so the website can be unavailable for a few minutes after the stack is complete.

### 6. Why does this homework update the EC2 tag instead of changing the webpage HTML?

User Data runs only on the first boot of an instance. If the HTML in the template changes, CloudFormation updates the stack, but the existing instance never re-runs the script, so the page stays the same. A tag change is applied directly by CloudFormation without replacing the instance. It clearly shows an update to the same stack with the same Instance ID.

### 7. What should happen to AWS resources when the assignment is finished?

They should be deleted. After the evidence is captured, both CloudFormation stacks (dev and test) are deleted in the console in us-east-1. CloudFormation removes the EC2 instance, VPC, subnet, internet gateway, route table and security group together. Then confirm the instance shows Terminated so no charges continue. Do not push to `main` again after cleanup, or the stack may be recreated. When the course ends, delete the IAM role and policy that are no longer needed.
