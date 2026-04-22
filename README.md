# BEM11 Lab 2 — Highly Available Multi-AZ VPC (AWS CloudFormation)

> Student: **Bernard Mawulorm Kofi Wodoame**
> Lab: **BEM11 Lab 2 — Highly Available Multi-AZ VPC**

This repository provisions a secure, fault-tolerant, multi-AZ Virtual Private
Cloud (VPC) in AWS entirely through Infrastructure as Code (CloudFormation).
The architecture implements a classic two-tier design (public web tier +
private application tier), with redundant NAT Gateways, and uses **AWS Systems
Manager Session Manager** as the *only* way to administer EC2 instances — no
SSH ports are ever opened, and no key pairs are distributed.

---

## 1. Architecture

```
                           ┌──────────────────────────┐
                           │        Internet          │
                           └────────────┬─────────────┘
                                        │
                             ┌──────────▼──────────┐
                             │  Internet Gateway   │
                             └──────────┬──────────┘
            VPC 10.20.0.0/16            │
 ┌──────────────────────────────────────┼──────────────────────────────────────┐
 │                                      │                                      │
 │  ┌───────────────────────────┐       │       ┌───────────────────────────┐  │
 │  │ Public Subnet AZ-a        │       │       │ Public Subnet AZ-b        │  │
 │  │ 10.20.1.0/24              │       │       │ 10.20.2.0/24              │  │
 │  │  • Web EC2 #1 (Apache)    │◄──────┴──────►│  • Web EC2 #2 (Apache)    │  │
 │  │  • NAT Gateway AZ-a       │               │  • NAT Gateway AZ-b       │  │
 │  └──────────────┬────────────┘               └──────────────┬────────────┘  │
 │                 │                                           │               │
 │  ┌──────────────▼────────────┐               ┌──────────────▼────────────┐  │
 │  │ Private Subnet AZ-a       │               │ Private Subnet AZ-b       │  │
 │  │ 10.20.11.0/24             │               │ 10.20.12.0/24             │  │
 │  │  • App EC2 #1             │               │  • App EC2 #2             │  │
 │  └───────────────────────────┘               └───────────────────────────┘  │
 │                                                                              │
 └──────────────────────────────────────────────────────────────────────────────┘
```

### Key design decisions

| Requirement | How it is met |
|---|---|
| Highly available, multi-AZ VPC | Two AZs, each with its own public + private subnet |
| Redundant public tier | `WebInstance1` in AZ-a, `WebInstance2` in AZ-b |
| Redundant private tier | `AppInstance1` in AZ-a, `AppInstance2` in AZ-b |
| Highly available NAT | One NAT Gateway *per AZ*, each with its own Elastic IP |
| Fault isolation | Each private subnet has its **own** route table pointing to its local NAT, so an AZ-level NAT failure does not impact the other AZ |
| Controlled internet access | Only public subnets route to the IGW; private subnets egress only through NAT |
| Secure EC2 access | No SSH rules in any security group; all instances have `AmazonSSMManagedInstanceCore` and are reached via Session Manager |
| Web tier security group | Ingress: TCP 80 from `0.0.0.0/0` only |
| App tier security group | Ingress: TCP 8080 only from the Web tier SG |
| Amazon Linux AMI | Resolved at deploy time via the SSM public parameter `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64` (region-agnostic) |
| Infrastructure as Code | Single CloudFormation template: `vpc-infrastructure.yaml` |

---

## 2. Repository contents

```
.
├── vpc-infrastructure.yaml   # The full CloudFormation template
├── README.md                 # This file
└── .gitignore
```

---

## 3. Prerequisites

* An AWS account with permissions to create VPCs, EC2, IAM roles, and CloudFormation stacks.
* AWS CLI v2 installed and configured (`aws configure`) **or** access to the AWS Console.
* The AWS Session Manager plugin for the AWS CLI (only needed if you want to
  open sessions from your local terminal):
  <https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html>

No SSH key pair is required.

---

## 4. Deploying the stack

### Option A — AWS CLI (recommended)

```bash
aws cloudformation deploy \
  --stack-name bem11-lab2-vpc \
  --template-file vpc-infrastructure.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      StudentFullName="Bernard Mawulorm Kofi Wodoame" \
      LabName="BEM11 Lab 2 - Highly Available Multi-AZ VPC"
```

The stack usually reaches `CREATE_COMPLETE` in 4–6 minutes (most of that time
is the two NAT Gateways).

### Option B — AWS Console

1. Open **CloudFormation → Create stack → With new resources**.
2. Upload `vpc-infrastructure.yaml`.
3. Name the stack `bem11-lab2-vpc`.
4. Accept the defaults or override `StudentFullName` / `LabName`.
5. Tick **"I acknowledge that AWS CloudFormation might create IAM resources."**
6. Click **Create stack**.

---

## 5. Verifying the deployment

### 5.1 Web tier (public)

After the stack completes, grab the web instance DNS names from the stack
outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name bem11-lab2-vpc \
  --query "Stacks[0].Outputs[?OutputKey=='WebInstance1PublicDns' || OutputKey=='WebInstance2PublicDns'].[OutputKey,OutputValue]" \
  --output table
```

Open both URLs in a browser (over plain HTTP). Each page must display:

* The full name (`Bernard Mawulorm Kofi Wodoame`)
* The lab name
* Which AZ served the response

### 5.2 SSM Session Manager access

List managed instances (they should appear in `Online` state within ~2 minutes
of launch):

```bash
aws ssm describe-instance-information \
  --query "InstanceInformationList[].[InstanceId,PingStatus,PlatformName]" \
  --output table
```

Start a session on any instance — **web or app tier** — using only its
instance ID:

```bash
# Web instance 1
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx

# App instance (in a private subnet, no public IP — still works)
aws ssm start-session --target i-yyyyyyyyyyyyyyyyy
```

You should land in an interactive shell as `ssm-user`. From an app-tier
instance you can confirm outbound internet via the NAT Gateway:

```bash
curl -s https://checkip.amazonaws.com
```

---

## 6. How the "no SSH" guarantee is enforced

1. **Security groups** — Neither `WebSecurityGroup` nor `AppSecurityGroup`
   contains an ingress rule for TCP 22. The web tier only accepts TCP 80 from
   the internet; the app tier only accepts TCP 8080 from the web tier SG.
2. **No key pairs** — No `KeyName` property is set on any `AWS::EC2::Instance`,
   so there is no SSH key material anywhere in the stack.
3. **SSM agent** — Amazon Linux 2023 ships with the SSM Agent pre-installed.
   The IAM instance profile attached to every instance grants
   `AmazonSSMManagedInstanceCore`, which is the minimum policy required for
   Session Manager. Because SSM uses an outbound connection from the agent to
   AWS endpoints, private-subnet instances can be managed over the NAT
   Gateway without any inbound ports being opened.

---

## 7. Tearing down

```bash
aws cloudformation delete-stack --stack-name bem11-lab2-vpc
aws cloudformation wait stack-delete-complete --stack-name bem11-lab2-vpc
```

This removes every resource the stack created — VPC, subnets, route tables,
NAT Gateways, Elastic IPs, EC2 instances, IAM role, and security groups —
leaving no lingering charges.

---

## 8. Parameters reference

| Parameter | Default | Purpose |
|---|---|---|
| `ProjectName` | `BEM11-Lab2-VPC` | Prefix used in every `Name` tag |
| `StudentFullName` | `Bernard Mawulorm Kofi Wodoame` | Shown on the web page |
| `LabName` | `BEM11 Lab 2 - Highly Available Multi-AZ VPC` | Shown on the web page |
| `VpcCidr` | `10.20.0.0/16` | VPC CIDR |
| `PublicSubnet1Cidr` / `PublicSubnet2Cidr` | `10.20.1.0/24` / `10.20.2.0/24` | Public subnets |
| `PrivateSubnet1Cidr` / `PrivateSubnet2Cidr` | `10.20.11.0/24` / `10.20.12.0/24` | Private subnets |
| `InstanceType` | `t3.micro` | EC2 size for every instance |
| `LatestAmazonLinux2023Ami` | SSM public parameter | Resolves the newest Amazon Linux 2023 AMI at deploy time |

---

## 9. License

This lab is submitted as coursework for **BEM11**. The template is free to
reuse for educational purposes.
