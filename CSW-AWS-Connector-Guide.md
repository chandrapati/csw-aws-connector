# Cisco Secure Workload — AWS Connector Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult the [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/secure-workload/index.html) — specifically [Configure and Manage Connectors → AWS Connector](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html) — for authoritative guidance.

## Table of Contents
1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Capabilities](#3-capabilities)
4. [Prerequisites](#4-prerequisites)
5. [Step A — AWS IAM Setup](#5-step-a--aws-iam-setup)
6. [Step B — Configure the AWS Connector in CSW](#6-step-b--configure-the-aws-connector-in-csw)
7. [Step C — VPC and Capability Configuration](#7-step-c--vpc-and-capability-configuration)
8. [EKS Integration](#8-eks-integration)
9. [Segmentation with AWS Security Groups](#9-segmentation-with-aws-security-groups)
10. [Verification](#10-verification)
11. [Limits](#11-limits)
12. [Troubleshooting](#12-troubleshooting)
13. [Related Resources](#13-related-resources)

---

## 1. Overview

The **AWS connector** in Cisco Secure Workload automatically ingests workload inventory and labels from **AWS VPCs**, streams **VPC flow logs**, and enforces segmentation policy using **AWS native Security Groups** — all without deploying CSW agents on every EC2 instance.

This is the foundation of CSW visibility and segmentation for AWS-hosted workloads, enabling consistent Zero Trust policy across on-premises and AWS environments from a single policy engine.

### What the AWS connector provides

| Capability | Description | Use case |
|------------|-------------|---------|
| **Label ingestion** | EC2 instance tags pulled into CSW labels | CMDB-style scope automation |
| **Flow log ingestion** | VPC flow logs from S3 → CSW for ADM | Traffic visualization, policy discovery |
| **Segmentation** | CSW policies → AWS Security Groups (auto-programmed) | Cloud-native enforcement |
| **EKS integration** | K8s node/pod/service metadata from EKS | Container workload visibility |

> **No virtual appliance required.** Unlike the NetFlow / ERSPAN / ISE connectors, AWS (and the other cloud connectors) do **not** run on a Secure Workload Ingest or Edge virtual appliance. Secure Workload connects to the AWS APIs directly.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         AWS Account / VPC                             │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  VPC (us-east-1)                                              │   │
│  │                                                               │   │
│  │  EC2 instances with tags:                                     │   │
│  │    Name=PaymentAPI, Env=Production, Team=Finance              │   │
│  │                                                               │   │
│  │  VPC Flow Logs ──────────────────────────► S3 Bucket          │   │
│  │                                            (flow-logs-csw)   │   │
│  │  Security Groups ◄── CSW programs policies                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                  ▲ EC2 API (tags)                                     │
│                  │ S3 API (flow logs)                                 │
│                  │ EC2 API (security groups)                          │
│                  │                                                    │
│  IAM User / Role with CloudFormation-provisioned permissions          │
└──────────────────────────────────────────────────────────────────────┘
                   ▲
                   │ HTTPS API calls
                   │
┌──────────────────┴───────────────────────────────────────────────────┐
│  Cisco Secure Workload (SaaS or On-Prem)                              │
│  AWS Connector (no virtual appliance needed)                          │
│                                                                       │
│  • Polls EC2 API for tag changes                                      │
│  • Reads VPC flow logs from S3                                        │
│  • Programs security groups when enforcement enabled                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Capabilities

### 3.1 Label Ingestion (always required)
- EC2 instance tags → CSW workload labels (synced continuously)
- Network interface tags → merged with instance labels
- Labels follow the format: AWS tag key = CSW label key

### 3.2 VPC Flow Log Ingestion (optional)
- Requires VPC flow logs enabled and published to **Amazon S3**
- CSW reads S3 bucket on a schedule (not CloudWatch — must be S3)
- Only **VPC-level** flow logs can be ingested
- Flow logs must capture **both Allowed and Denied** traffic
- Hourly and daily flow-log partitions are both supported
- Required flow log attributes (any others are ignored): Source Address, Destination Address, Source Port, Destination Port, Protocol, Packets, Bytes, Start Time, End Time, Action, TCP Flags, Interface-ID, **Log status**, Flow Direction, **Packet Source Address**, **Packet Destination Address**

### 3.3 Segmentation (optional, requires label ingestion)
- When segmentation is enabled for a VPC, CSW programs the policies as **AWS native Security Groups**; relevant policies are programmed automatically once enforcement is enabled
- AWS Security Groups support **ALLOW rules only** — your segmentation policies should be Allow policies, with the **Catch-All set to Deny**
- **Warning:** Enabling segmentation for a VPC **removes/overwrites the existing Security Groups** on that VPC. CSW automatically backs up the SGs first (see §9); still take your own independent backup as defense-in-depth
- Requires label ingestion (Gather Labels) to be enabled

### 3.4 EKS Integration (optional)
- Collects node, pod, and service metadata from EKS clusters
- Private EKS clusters require **Secure Connector** (CSW agentless proxy)

---

## 4. Prerequisites

### AWS account requirements
- [ ] AWS user (IAM user or role) dedicated to the CSW connector
- [ ] CloudFormation permissions to upload the CFT generated by the CSW connector wizard
- [ ] For flow logs: VPC flow logs enabled, published to S3; S3 bucket accessible by the connector IAM user
- [ ] For segmentation: Existing Security Group rules backed up
- [ ] For EKS: EKS cluster accessible by the connector IAM user; Secure Connector if private cluster

### CSW requirements
- [ ] CSW cluster administrator access (to create connectors)
- [ ] Target VRF/tenant configured for the AWS workloads
- [ ] Each VPC belongs to exactly **one** AWS connector

---

## 5. Step A — AWS IAM Setup

The CSW connector wizard generates a **CloudFormation Template (CFT)** that provisions the exact IAM permissions needed. This is the recommended approach.

### A1 — Start the connector wizard in CSW

1. CSW UI: **Manage > Workloads > Connectors**, then choose the **AWS** cloud connector and start the wizard
2. The wizard will ask which capabilities you want to enable (Gather Labels / Ingest Flow Logs / Segmentation / EKS). **Recommended for a POV: enable Gather Labels + Ingest Flow Logs now, and leave Segmentation off** until you are ready to enforce (see §9)
3. Based on the capabilities selected, CSW generates the matching **CloudFormation Template (CFT)** — download it

### A2 — Apply the CFT in AWS

**Option A — AWS Console:**
1. Log into AWS Console → **CloudFormation > Stacks > Create Stack**
2. Upload the downloaded CFT
3. Follow the wizard to create the stack
4. Note the IAM user or role ARN created

**Option B — AWS CLI:**
```bash
aws cloudformation deploy \
  --template-file csw-connector-permissions.yaml \
  --stack-name csw-connector-iam \
  --capabilities CAPABILITY_IAM
```

### A3 — Role-based authentication (recommended over access keys)

CSW supports IAM **role-based authentication** to avoid long-lived credential keys:
1. In the connector wizard, click the **Role** tab
2. CSW will display an External ID and User ARN
3. Create an IAM role in AWS with the connector's IAM permissions
4. Set the trust relationship to allow the CSW User ARN to assume the role
5. Provide the Role ARN in the CSW connector wizard

> Role-based authentication is preferred over IAM user access keys for security — no long-lived credentials stored.

### A4 — For cross-account access

To monitor VPCs in multiple AWS accounts:
1. Create the connector IAM role in a central account
2. In each additional account, create a role that the central account role can assume
3. Configure cross-account trust relationships per the AWS cross-account access documentation

---

## 6. Step B — Configure the AWS Connector in CSW

### B1 — Connector basics

| Field | Value |
|-------|-------|
| **Connector Name** | e.g., `aws-prod-connector` |
| **Authentication** | Role-based (ARN) or Access Key / Secret |
| **AWS Region** | Primary region of your VPCs |

### B2 — Credential entry

**Role-based (recommended):**
| Field | Value |
|-------|-------|
| Role ARN | `arn:aws:iam::123456789012:role/CSW-Connector-Role` |
| External ID | (auto-populated by CSW) |

**Key-based (alternative):**
| Field | Value |
|-------|-------|
| Access Key ID | IAM user access key |
| Secret Access Key | IAM user secret |

### B3 — VPC selection

After credentials are entered, CSW discovers all VPCs in the configured region:
1. Select the VPCs to integrate
2. For each VPC, choose which capabilities to enable

---

## 7. Step C — VPC and Capability Configuration

For each selected VPC:

### Labels / Inventory
- **Always enable** — required by other capabilities
- CSW polls EC2 API for tag changes continuously

### Flow Log Ingestion
If enabling flow log ingestion:
1. Ensure VPC flow logs are enabled and publishing to S3
2. Configure VPC flow logs in AWS (if not already done):
   ```bash
   aws ec2 create-flow-logs \
     --resource-type VPC \
     --resource-ids vpc-xxxxxxxx \
     --traffic-type ALL \
     --log-destination-type s3 \
     --log-destination arn:aws:s3:::your-flow-log-bucket \
     --log-format "${version} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${tcp-flags} ${interface-id} ${log-status} ${flow-direction} ${pkt-srcaddr} ${pkt-dstaddr}"
   ```
3. Provide the S3 bucket name in the CSW connector VPC configuration

### Segmentation
> **Best practice (per Cisco): do _not_ enable Segmentation during initial configuration.** Gather labels and flows first, build and analyze policies in a workspace, enable enforcement on that workspace, and only then return to the connector to enable Segmentation for the VPC. See §9.

If/when you enable segmentation:
1. Confirm **Gather Labels** is enabled (required dependency)
2. Take your own backup of the VPC's Security Groups (CSW also auto-backs them up — see §9)
3. **Enable enforcement in the workspace _before_ enabling Segmentation for the VPC.** If you enable Segmentation on a VPC that is not in an enforcement-enabled workspace, **all traffic on that VPC is allowed**
4. Edit the connector and enable Segmentation for the target VPC — CSW then takes ownership of the Security Groups

### Apply
Click **Test and Apply**. The connector begins:
- Pulling EC2 tags and registering workloads in CSW inventory
- Reading VPC flow logs from S3 (if configured)
- Monitoring for policy changes to push to Security Groups (if segmentation enabled)

---

## 8. EKS Integration

When EKS clusters are running in your VPC:

1. Enable the **Kubernetes** option in the VPC configuration
2. CSW collects:
   - Node IP addresses and labels
   - Pod IP addresses, namespaces, and labels
   - Service cluster IPs and port mappings
3. For **private EKS clusters**: deploy the **Secure Connector** to allow CSW API access through private networking

K8s metadata appears in CSW inventory as labels:
```
pod/name = payment-api-6b9f7d
pod/namespace = production
pod/app = payment-api
node/name = ip-10-0-1-55
```

---

## 9. Segmentation with AWS Security Groups

When enforcement is enabled in CSW for an AWS VPC:

### How it works
1. You define policies in CSW using label-based scopes (no IP addresses required)
2. CSW translates policies → AWS Security Group rules
3. Security Groups are automatically applied to EC2 instances and ENIs in the VPC
4. As workloads start/stop or policies change, CSW updates Security Groups automatically

### Pre-requisites and ordering (important)
- Enabling Segmentation for a VPC **removes the existing Security Groups** on that VPC and replaces them with CSW-programmed SGs
- **Enable enforcement in the workspace _first_.** Enabling Segmentation on a VPC that is not part of an enforcement-enabled workspace results in **all traffic being allowed** on that VPC
- AWS Security Groups are **allow-list only** — author Allow policies and rely on a **Catch-All = Deny** to block everything else. There is no per-policy "deny" rule in AWS SGs

### Automatic backup and restore of Security Groups
CSW automates SG backup/restore around the segmentation toggle:
- **Automatic backup** — when you enable Segmentation for a VPC, CSW automatically backs up that VPC's current Security Groups (runs in the background, no manual action). Only one backup state is kept per VPC; a re-backup occurs if VPCs change, and restores always use the **most recent** backup state
- **Automatic restore** — when you **disable** Segmentation for the VPC, CSW restores the SGs **it modified** to that most-recent backup state; unrelated SG configuration is left untouched
- Toggle via **Manage > Workloads > Connectors > AWS Connector > Resources > Resources Tree**, then select the VPC and set the **Segmentation** radio button
- Restore only applies to VPCs where Segmentation was enabled and then disabled
- Still take an **independent backup** of your SGs as defense-in-depth before your first enforcement test

### Policy example
```
Consumer: Dev-VMs    (scope: tag/Environment = Development)
Provider: Prod-DBs   (scope: tag/Environment = Production AND tag/Tier = Database)
Action:   ALLOW TCP 1433   (only the flows you intend to permit)
Catch-All (scope default): DENY
```
CSW programs Security Groups that permit only the explicitly-allowed flows; everything else (including Dev-VMs → Prod-DBs on any other port) is dropped by the Catch-All — enforced natively by AWS.

---

## 10. Verification

### Check connector status
**Manage > Workloads > Connectors**, then select the AWS connector. Review the per-VPC rows; the connector should be syncing with a recent timestamp.

### Check inventory enrichment
1. On the AWS connector page, click a **VPC** row, then click an **IP address** to open its **Inventory Profile**
2. Confirm AWS tags appear as labels (see *Labels Generated by Cloud Connectors*)

### Check flow data (if enabled)
1. Use **Investigate > Traffic** (Flow Search) and filter by the VPC CIDR
2. Confirm flows from the VPC are visible (allow ~10–15 min after flow logs land in S3)

### Test segmentation (if enabled)
1. Confirm enforcement is enabled on the workspace, then enable Segmentation for the VPC
2. Check enforcement state under **Defend > Enforcement Status** (see *Enforcement Status for Cloud Connectors*)
3. In the AWS Console → EC2 → Security Groups, confirm CSW-managed rules are present

---

## 11. Limits

| Metric | Limit |
|--------|-------|
| VPCs per AWS connector | Multiple (one connector can manage multiple VPCs) |
| AWS connectors per CSW cluster | Multiple |
| Connectors per VPC | Exactly one — a VPC can belong to only one AWS connector |
| Flow log source | **S3 only** — CSW cannot collect flow data from CloudWatch Logs |
| Flow log scope | VPC-level flow logs only |
| Flow log partition | Hourly and daily supported |
| EKS private cluster | Requires Secure Connector |

---

## 12. Troubleshooting

| Symptom | Check |
|---------|-------|
| Labels not syncing | Verify IAM permissions (EC2:DescribeInstances, EC2:DescribeTags); check connector logs |
| Flow logs not appearing | Confirm VPC flow logs enabled → S3; S3 bucket accessible by IAM user; flow log format includes required fields |
| Segmentation not enforcing | Confirm Gather Labels enabled and the workspace has enforcement enabled; check IAM has EC2:AuthorizeSecurityGroupIngress/Egress permissions |
| VPC unexpectedly allows all traffic | Segmentation was enabled on a VPC whose workspace is not enforcing, or the Catch-All policy is not set to **Deny** |
| Concrete policy shows **SKIPPED** | The generated rule count exceeds AWS Security Group limits (AWS counts each unique subnet × unique port as a rule). Consolidate policies (e.g., larger subnets); contact AWS before raising SG limits, then update the limit in the connector config |
| EKS pods not visible | Confirm EKS IAM permissions in CFT and the user/role is in the `aws-auth` ConfigMap; for private EKS, deploy Secure Connector |
| Role auth failing | Verify trust relationship between CSW User ARN and the connector role; check External ID matches |

---

## 13. Related Resources

| Repository | Description | Best for |
|------------|-------------|---------|
| [csw-azure-connector](https://github.com/chandrapati/csw-azure-connector) | Azure VNet label ingestion, NSG enforcement | Azure workloads |
| [csw-gcp-connector](https://github.com/chandrapati/csw-gcp-connector) | GCP VPC label ingestion, firewall enforcement | GCP workloads |
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents inside EC2 for deep visibility | Agent-based visibility |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery → enforcement workflow | Policy management |
| [csw-splunk-integration](https://github.com/chandrapati/csw-splunk-integration) | CSW syslog alerts → Splunk | SecOps alerting |
| [CSW-Operations-Toolkit](https://github.com/chandrapati/CSW-Operations-Toolkit) | Day-2 ops: health checks, reporting | Ongoing operations |

> **Cloud strategy tip:** Deploy AWS + Azure + GCP connectors together in a single CSW tenant for unified multi-cloud policy management with a consistent label taxonomy.

---
*Community reference — Cisco Solutions Engineering. Not an official Cisco product document.*
