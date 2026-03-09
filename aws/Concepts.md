# AWS: Concepts

> **The Anvaya:** *Every AWS service is either a compute primitive, a network primitive, a storage primitive, or an identity primitive — and every security boundary is enforced by IAM.*

---

## **<a id="region-az"></a>Region & Availability Zone**

> A Region is a geographic cluster of data centers. An Availability Zone (AZ) is one isolated data center within a Region.

- Each Region (`us-east-1`, `ap-south-1`) is completely independent. No data flows between regions unless you configure it.
- Each Region has 2–6 AZs (e.g., `ap-south-1a`, `ap-south-1b`).
- AZs within a Region are connected by low-latency fiber — cross-AZ network latency is ~1ms.
- **Deploying across 2+ AZs** = high availability. One AZ failure leaves your service running.

---

## **<a id="account"></a>AWS Account**

> An isolated billing and authorization boundary. Every resource belongs to exactly one account.

- Account ID is a 12-digit number (e.g., `123456789012`). It appears in every ARN.
- **AWS Organizations:** Groups multiple accounts under a master "management account." Enables Service Control Policies (SCPs) that restrict what any IAM entity in the account can do — even admins.
- Use separate accounts per environment (dev / staging / prod) for blast radius control. A mistake in dev cannot affect prod resources in a different account.

---

## **<a id="arn"></a>ARN (Amazon Resource Name)**

> A globally unique string that identifies any AWS resource.

Format: `arn:partition:service:region:account-id:resource`

- `arn:aws:s3:::my-bucket` (S3 is global — no region/account)
- `arn:aws:ec2:ap-south-1:123456789:instance/i-0abc123`
- `arn:aws:iam::123456789:role/my-role` (IAM is global — no region)
- Used everywhere: IAM policies, resource-based policies, CloudWatch alarms, ECS task definitions.

---

## **<a id="iam-user"></a>IAM User**

> A long-term identity for a human with a username, password, and optional access keys.

- Access keys (`AKIA...` + secret) are long-lived credentials — never embed in code.
- **MFA** should be required for all human IAM users.
- Best practice (2024+): Use **IAM Identity Center** (SSO) for human access instead of long-lived IAM user keys. Provides temporary credentials that automatically expire.
- No more than 2 access keys per user exist at any time (to enable zero-downtime rotation).

---

## **<a id="iam-role"></a>IAM Role**

> A short-term identity that any AWS service, EC2 instance, Lambda, or cross-account entity can assume to get temporary credentials.

- Roles have two types of policies attached:
  - **Trust Policy:** Who can assume this role (`sts:AssumeRole`)?
  - **Permission Policy:** What can the role do once assumed?
- When a role is assumed, the caller gets temporary credentials that expire (15 min to 12 hours).
- EC2 instances → **Instance Profile** (wrapper around a role)
- ECS tasks → `taskRoleArn`
- EKS pods → **IRSA** (IAM Roles for Service Accounts)
- GitHub Actions → OIDC role assumption (no credentials stored)

---

## **<a id="iam-policy"></a>IAM Policy**

> A JSON document defining Allow or Deny for specific Actions on specific Resources.

```json
{
  "Effect": "Allow | Deny",
  "Action": ["s3:GetObject"],        // API call name: <service>:<Action>
  "Resource": "arn:aws:s3:::bucket/*",
  "Condition": { ... }               // Optional: restrict by time, IP, MFA, tag
}
```

- **Managed policies:** AWS-owned (`AmazonS3ReadOnlyAccess`) or customer-created. Reusable across roles.
- **Inline policies:** Embedded directly on a single role/user. Useful for non-shareable, tightly-coupled permissions.
- **Deny always wins** over Allow. An explicit Deny can never be overridden by an Allow.
- **IAM Access Analyzer:** Scans your policies for overly permissive access and highlights externally accessible resources.

---

## **<a id="vpc"></a>VPC (Virtual Private Cloud)**

> A logically isolated private network within AWS where you launch your resources.

- CIDR block defines the IP range: `10.0.0.0/16` = 65,536 IPs.
- Resources in a VPC can communicate privately. Internet access requires explicit configuration.
- VPCs are region-scoped. You can't span a VPC across regions.
- **VPC Peering** connects two VPCs privately (same or different accounts). Non-transitive — A-B and B-C peering does NOT give A access to C.
- **VPC Endpoints** let resources in private subnets access AWS services (S3, DynamoDB, Secrets Manager) without going through a NAT Gateway.

---

## **<a id="subnet"></a>Subnet**

> A range of IPs within a VPC, associated to exactly one AZ.

- **Public subnet:** Route table has a route to an Internet Gateway. Suitable for internet-facing resources.
- **Private subnet:** Route table has a route to a NAT Gateway (for outbound internet). Resources not directly reachable from internet.
- **Isolated subnet:** No internet route. For databases.
- Rule of thumb: 5 IPs per subnet are reserved by AWS (first 4 + last 1). A `/24` gives you 251 usable IPs.

---

## **<a id="internet-gateway"></a>Internet Gateway (IGW)**

> The VPC component that enables internet traffic in and out of public subnets.

- Attached at the VPC level (one per VPC). Horizontally scalable — no bandwidth limit.
- A subnet is "public" only when its route table has: `0.0.0.0/0 → igw-xxx`.
- Resources also need a public IP or Elastic IP to receive inbound internet traffic.

---

## **<a id="nat-gateway"></a>NAT Gateway**

> Allows resources in private subnets to initiate outbound connections to the internet while remaining unreachable from the internet.

- Private subnet resources route `0.0.0.0/0` to the NAT Gateway, which is in the public subnet.
- NAT Gateway is AZ-specific. For HA, deploy one in each AZ and update each private subnet's route table.
- **Cost:** ~$0.045/hour + $0.045/GB processed. For heavy egress workloads, consider **VPC Endpoints** to bypass NAT for AWS service traffic.

---

## **<a id="security-group"></a>Security Group**

> A stateful, virtual firewall attached to network interfaces (EC2, RDS, ALB, Lambda, etc.).

- **Stateful:** If inbound traffic is allowed, the response is automatically allowed. You don't need to explicitly allow outbound for responses.
- Rules are **additive** (Allow only). Cannot explicitly Deny — that's what NACLs are for.
- You can reference another Security Group as a source/destination — best practice for tiered architectures:
  - DB SG allows inbound 5432 from *App SG* (not from a CIDR). As app servers scale, new instances automatically get access.
- Default: all outbound allowed, all inbound denied.

---

## **<a id="nacl"></a>Network ACL (NACL)**

> A stateless firewall at the subnet boundary. Evaluated before traffic reaches the instance.

- **Stateless:** You must allow both inbound AND outbound for a connection (return traffic needs an explicit rule).
- Rules are numbered and evaluated in order. First match wins. Default rule (`*`) denies all.
- Use NACLs for **subnet-level blocking** (e.g., block an attacker's IP range). Use Security Groups for instance-level control.
- Most teams rarely customize NACLs — Security Groups are sufficient for most workloads.

---

## **<a id="ec2"></a>EC2 (Elastic Compute Cloud)**

> A virtual machine running on AWS infrastructure. Billed per second (minimum 60 seconds).

- **Instance type:** Family + size (e.g., `m6i.large` = general-purpose, Intel, 2 vCPU, 8 GB RAM).
- **AMI (Amazon Machine Image):** Snapshot of an OS + pre-installed software. Used to launch new instances.
- **Elastic IP:** A static public IPv4 address you can attach to an instance. Charged when *not* attached (incentivizes not wasting IPs).
- **Placement Groups:** Cluster (pack instances close together for low latency), Spread (one instance per AZ rack for isolation), Partition (groups of instances for distributed systems like Kafka).

---

## **<a id="asg"></a>Auto Scaling Group (ASG)**

> A managed group of EC2 instances that automatically scales in/out based on demand or schedules.

- Minimum, desired, and maximum instance counts define the scale bounds.
- **Launch Template:** The "blueprint" — which AMI, instance type, SG, IAM role, user-data to use for new instances.
- Integrates with ALB to automatically register/deregister instances in target groups.
- Lifecycle hooks let you run scripts before an instance enters or leaves service (drain connections, snapshot).

---

## **<a id="s3"></a>S3 (Simple Storage Service)**

> Object storage for any amount of data. 11 nines of durability (99.999999999%).

- **Bucket:** Container for objects. Globally unique name. Single region.
- **Object:** File + metadata. Max size 5 TB. Accessed via key (file path-like string).
- **Storage Classes:** `STANDARD` (frequent access) → `STANDARD_IA` (infrequent, cheaper storage) → `GLACIER` (archival, cheap, slow retrieval). Lifecycle rules automate transitions.
- **Versioning:** Keeps every version of every object. Required for MFA Delete protection and Terraform backends.
- **Presigned URLs:** Time-limited URLs for temporary access to private objects — generate server-side, share with clients.

---

## **<a id="ecr"></a>ECR (Elastic Container Registry)**

> A managed Docker container registry integrated with AWS IAM and ECS/EKS.

- Images authenticated via `aws ecr get-login-password` — no static Docker credentials.
- ECR tokens expire every 12 hours. CI pipelines must refresh before pushing.
- **Image scanning:** `BASIC` (free, CVE database) or `ENHANCED` (powered by Inspector, paid).
- ECR Public Gallery: public registries at `public.ecr.aws` — free pull for public images.

---

## **<a id="ecs"></a>ECS (Elastic Container Service)**

> AWS-managed container orchestration. Run Docker containers without managing Kubernetes.

- **Cluster:** Logical grouping of tasks/services.
- **Task Definition:** Container spec (image, CPU, memory, ports, env vars, secrets, log config). Versioned — each update creates a new revision.
- **Task:** A running instance of a Task Definition. Ephemeral.
- **Service:** Maintains N running copies of a Task Definition. Integrates with ALB for traffic routing.
- **Launch Types:** `FARGATE` (serverless — no EC2 to manage) or `EC2` (you manage the host EC2 instances for more control).

---

## **<a id="eks"></a>EKS (Elastic Kubernetes Service)**

> AWS-managed Kubernetes control plane. AWS handles API server, etcd, and control plane upgrades.

- You manage Node Groups (EC2 worker nodes) or use Fargate Profiles (serverless).
- **Managed Node Groups:** AWS automates node OS patching and safe rollouts on upgrade.
- **Cluster add-ons:** VPC CNI, CoreDNS, kube-proxy, EBS CSI Driver — managed and upgradeable separately.
- **Access Entry:** The modern way to map IAM roles to Kubernetes RBAC roles (replaced `aws-auth` ConfigMap).

---

## **<a id="irsa"></a>IRSA (IAM Roles for Service Accounts)**

> The mechanism that lets individual Kubernetes pods assume specific IAM roles without any node-level credentials.

- Works via the **OIDC Provider** configured on the EKS cluster. The pod requests a projected service account token, Amazon STS validates it against the OIDC provider, and returns AWS credentials.
- Annotation on the Kubernetes ServiceAccount: `eks.amazonaws.com/role-arn: arn:aws:iam::123:role/my-role`
- Pod spec references `serviceAccountName: my-sa` — the pod gets credentials via the `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` projected file.
- The IAM Role trust policy must allow the OIDC provider and specific service account as principal.

---

## **<a id="alb"></a>ALB (Application Load Balancer)**

> An HTTP/HTTPS Layer 7 load balancer that routes traffic based on path, hostname, headers, or query params.

- Lives in public subnets across 2+ AZs. Internet-facing or internal.
- **Listener:** Port + protocol. HTTPS listener requires an ACM certificate.
- **Target Group:** The set of backend targets (EC2 instances, ECS tasks by IP, Lambda functions). Each target group has a health check.
- **Listener Rules:** Conditions (path `/api/*`, host `api.example.com`) → action (forward to target group, redirect, return fixed response).
- `X-Forwarded-For` header carries the original client IP. Your app should read this for access logs.

---

## **<a id="rds"></a>RDS (Relational Database Service)**

> Managed relational databases: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server. AWS handles patching, backups, failover.

- **DB Instance:** The compute + storage unit. Instance class controls CPU/RAM (`db.t3.medium`, `db.m6g.large`).
- **Multi-AZ:** Synchronous standby replica in a second AZ. Automatic failover (~60s) on primary failure. Not a read replica — standby is passive.
- **Read Replicas:** Asynchronous replicas for read scaling and (after promotion) disaster recovery. Up to 5 per instance.
- **DB Subnet Group:** RDS requires you to designate subnets in 2+ AZs to place the primary + standby.
- Automated backups to S3 (point-in-time recovery up to retention period). Manual snapshots persist until deleted.

---

## **<a id="secrets-manager"></a>Secrets Manager**

> Encrypted secret storage with built-in rotation for AWS-managed services.

- Secret types: `SecretString` (JSON or plain string), `SecretBinary` (binary data).
- Rotation via Lambda functions. AWS provides pre-built rotation functions for RDS, Redshift, DocumentDB.
- Apps read secrets at runtime via SDK — secret values are never baked into AMIs or images.
- **Secret versions:** `AWSCURRENT`, `AWSPENDING` (during rotation), `AWSPREVIOUS`. Your app always reads `AWSCURRENT`.

---

## **<a id="ssm-parameter-store"></a>SSM Parameter Store**

> Hierarchical key-value store for config and secrets. Free for Standard tier.

- Standard parameters (≤4 KB): free. Advanced parameters (≤8 KB, policies): $0.05/parameter/month.
- `SecureString` type: encrypted with KMS.
- Hierarchy: `/myapp/prod/db-host` — IAM policies can grant access to an entire prefix (`/myapp/prod/*`).
- ECS can inject SSM parameters as container environment variables natively (same as Secrets Manager).

---

## **<a id="cloudwatch"></a>CloudWatch**

> AWS's unified observability service: logs, metrics, alarms, dashboards, and synthetics.

- **Log Groups:** Container for log streams. Set retention or logs are kept forever (billed monthly per GB).
- **Log Insights:** SQL-like query language for CloudWatch Logs. Useful for ad-hoc debugging.
- **Metrics:** Time-series data points. AWS services emit default metrics. Custom metrics cost $0.30/metric/month.
- **Alarms:** Trigger SNS notifications, Auto Scaling actions, or EC2 instance recovery based on metric thresholds.
- **Container Insights:** Enhanced metrics (CPU, memory, network, disk) for ECS and EKS at per-node/task cost.

---

## **<a id="cloudtrail"></a>CloudTrail**

> Records every API call made in your AWS account — who did what, when, from where.

- Enabled by default but only retains 90 days. Create a **Trail** to write to S3 for long-term retention.
- Every API call: `who (principal ARN)` + `what (action)` + `when (timestamp)` + `from where (IP)` + `which resource`.
- Indispensable for security incidents ("who deleted that S3 bucket?") and compliance audits.
- Enable **CloudTrail Insights** to detect unusual API call rates (potential attacks).

---

## **<a id="route53"></a>Route53**

> AWS's managed DNS service. Globally distributed, authoritative DNS.

- **Hosted Zone:** Container for DNS records for a domain. Comes in Public (internet-facing) and Private (VPC-internal, e.g., `db.internal`).
- **Alias Records:** AWS-specific DNS feature. Points to AWS resources (ALB, CloudFront, S3) — no TTL charge, resolves to the resource's current IP(s) automatically.
- **Health Checks + Routing Policies:** Route 53 can do latency-based routing, geolocation routing, weighted round-robin, and failover — based on health check results.

---

## **<a id="acm"></a>ACM (AWS Certificate Manager)**

> Free TLS/SSL certificates that auto-renew and integrate directly with ALB, CloudFront, and API Gateway.

- **DV (Domain Validation) certificates** are free and issued in minutes.
- Certificates validated via DNS (recommended — add a CNAME record) or email.
- ACM certificates can **only be used with AWS services** — you cannot export the private key. For self-managed servers, use Let's Encrypt instead.
- Certificates used with CloudFront or API Gateway must be in `us-east-1` (even if your resources are in another region).
