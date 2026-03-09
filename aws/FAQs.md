# AWS: FAQs

> **The Anvaya:** *Most AWS confusion resolves when you understand the authorization model: every API call is an IAM check, and every network call is a Security Group check.*

---

## 🏛️ IAM & Auth

**Q: My EC2 instance / ECS task / Lambda is getting `Access Denied` on an AWS API call. How do I debug?**

The error message usually includes the principal ARN and the failed action. Follow this checklist:

1. **Does the role exist and is it attached?** `aws iam get-instance-profile --instance-profile-name <name>` or check ECS Task Definition `taskRoleArn`.
2. **Does the role's policy allow the action?** Use IAM Policy Simulator: IAM → Policy Simulator → select role → test the action.
3. **Is there a resource-based policy denying it?** (e.g., S3 bucket policy, KMS key policy). An explicit Deny in *any* policy trumps all Allows.
4. **Is the resource in the correct region?** `ap-south-1` role won't automatically work on `us-east-1` resources unless the ARN matches.

**Q: What's the difference between an IAM Role and an IAM User?**

| | IAM User | IAM Role |
| :--- | :--- | :--- |
| Credentials | Long-term (access key + secret) | Short-term (expire in minutes/hours) |
| Identity for | Humans | AWS services, apps, cross-account |
| Rotation | Manual | Automatic |
| Production use | Only with MFA; prefer SSO instead | Yes — always for workloads |

**Q: I need to give an external vendor access to my AWS account. How?**

Use a **cross-account IAM Role** — never share your account credentials.

1. Create an IAM Role in your account with a trust policy allowing the vendor's AWS account ID.
2. Give the vendor the Role ARN.
3. The vendor assumes the role with `sts:AssumeRole` to get temporary credentials.
4. When the engagement ends, delete the role. Done. No credentials to revoke.

```json
// Trust policy in YOUR account:
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::<VENDOR_ACCOUNT_ID>:root" },
    "Action": "sts:AssumeRole",
    "Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }
  }]
}
```

---

## 🏛️ Networking

**Q: I can't SSH into my EC2 instance. What do I check?**

Go through this checklist in order:

1. **Security Group:** Does the instance's SG have inbound rule for TCP 22 from your IP? (Not `0.0.0.0/0` — from *your specific IP*.)
2. **Subnet:** Is the instance in a public subnet with an Internet Gateway route? (Private subnet instances have no public IP and never receive inbound traffic.)
3. **Public IP:** Does the instance have a public IP or Elastic IP assigned?
4. **Key pair:** Are you using the correct `.pem` key? `ssh -i ~/.ssh/my-key.pem ec2-user@<public-ip>`
5. **OS firewall:** Is `iptables`/`firewalld` on the instance blocking port 22?
6. **Better approach:** Use SSM Session Manager instead — no port 22, no key management, works on private instances.

**Q: What's the difference between a Security Group and a NACL?**

| | Security Group | NACL |
| :--- | :--- | :--- |
| Level | Instance / ENI | Subnet |
| Stateful? | Yes (return traffic auto-allowed) | No (must allow inbound + outbound) |
| Rules | Allow only | Allow and Deny |
| Order | All rules evaluated | Numbered, first match wins |
| Use for | Instance-level access control (98% of cases) | Subnet-level explicit Deny (attacker IP block) |

For most architectures, Security Groups are sufficient. NACLs are a last resort for subnet-wide explicit denies.

**Q: My private subnet EC2 instance can't reach the internet (apt update fails). Why?**

Private subnets route internet traffic to a **NAT Gateway**. Check:

1. Does the private subnet's **route table** have `0.0.0.0/0 → nat-xxx`? (Not the Internet Gateway.)
2. Is the NAT Gateway in a **public subnet** (not private)?
3. Is the NAT Gateway in the **same AZ** as the private subnet, or using the correct cross-AZ NAT?
4. Does the instance's Security Group allow outbound traffic? (Default outbound SG rule allows all — check if it was modified.)

**Q: Should I put my ALB in a public or private subnet?**

- **Internet-facing ALB:** Must be in **public subnets** (the ALB itself needs public IPs).
- **Internal ALB:** Use **private subnets**. Internal ALBs have private IPs only — for service-to-service traffic within the VPC.
- Your ECS tasks / EC2 instances behind the ALB always go in **private subnets**, regardless of ALB type.

---

## 🏛️ ECS & EKS

**Q: My ECS task keeps stopping immediately after starting. Why?**

Check the task's **Stopped Reason** in the ECS Console (Cluster → Tasks → stopped task → Stopped Reason). Common causes:

1. **Container exited non-zero:** Application crashed on startup. Check CloudWatch logs for the container.
2. **OOM killed:** Container exceeded its memory limit. Increase `memory` in Task Definition.
3. **Image pull failed:** ECR auth failed or image tag doesn't exist. Verify `executionRoleArn` has `ecr:GetAuthorizationToken` + `ecr:BatchGetImage`.
4. **Secrets injection failed:** `valueFrom` ARN in Task Definition is wrong, or `executionRoleArn` lacks Secrets Manager permissions.

**Q: My EKS pod can't access AWS services (S3, Secrets Manager). How do I grant it access?**

Use **IRSA** [[?](Concepts.md#irsa)]. Step-by-step:

```bash
# 1. Create an IAM role with the needed permissions
# 2. Set the trust policy to allow the specific Kubernetes service account via OIDC
eksctl create iamserviceaccount \
  --name myapp-sa \
  --namespace production \
  --cluster myapp-prod \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# 3. Reference the service account in the pod spec
# spec:
#   serviceAccountName: myapp-sa
```

If the pod still fails: check `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` env vars are present in the pod (`kubectl exec` → `env | grep AWS`). If not, the service account annotation (`eks.amazonaws.com/role-arn`) is missing.

**Q: ECS vs EKS — which should I choose?**

| | ECS Fargate | EKS |
| :--- | :--- | :--- |
| Operations | Near zero (AWS managed) | Significant (node management, add-ons, upgrades) |
| Learning curve | Low | High (need Kubernetes knowledge) |
| Ecosystem | AWS-only | Portable, rich Helm/operator ecosystem |
| Cost | Pay per task CPU/memory | Pay per control plane ($0.10/hr) + nodes |
| Best for | Small-medium teams deploying containers on AWS | Teams with K8s expertise or multi-cloud plans |

Start with ECS. Migrate to EKS when you hit ECS limitations (Kubernetes-native tooling, complex scheduling, Helm charts your team depends on).

---

## 🏛️ Storage & Databases

**Q: My app needs to share a filesystem between multiple ECS tasks. What do I use?**

Use **Amazon EFS (Elastic File System)**. Mount it as a volume in the ECS Task Definition. EFS scales automatically, works across AZs, and persists data beyond task lifetime. Not suitable for high-throughput random I/O (use EBS for that). Cost: ~$0.30/GB/month.

```json
// ECS Task Definition volume:
"volumes": [{ "name": "shared-data", "efsVolumeConfiguration": { "fileSystemId": "fs-xxx" } }],
"mountPoints": [{ "containerPath": "/shared", "sourceVolume": "shared-data" }]
```

**Q: My RDS instance ran out of storage and crashed. How do I prevent this?**

Enable **Storage Autoscaling** when creating the RDS instance (`--max-allocated-storage 500`). AWS automatically scales storage when free space drops below 10% — no downtime. Also set a `FreeStorageSpace` CloudWatch alarm at 20% of your current storage.

**Q: What's the difference between RDS Multi-AZ and Read Replicas?**

| | Multi-AZ | Read Replica |
| :--- | :--- | :--- |
| Replication | Synchronous | Asynchronous (slight lag) |
| Purpose | High availability / failover | Read scaling / lower latency reads |
| Readable? | No (standby is passive) | Yes (separate endpoint) |
| Failover | Automatic (~60s) | Manual (promote to standalone) |
| Cost | 2× instance cost | Additional instance cost |

Use **Multi-AZ for all production databases**. Add Read Replicas only if read throughput demands it.

---

## 🏛️ Cost & Billing

**Q: My AWS bill is unexpectedly high. What are the biggest cost drivers?**

In order of typical impact:

1. **NAT Gateway data processing** ($0.045/GB). Fix: VPC endpoints for S3/DynamoDB remove this cost entirely for those services.
2. **EC2 running 24/7** when only needed 8 hours/day. Fix: Scheduled scaling or switch to Fargate (pay per task-second).
3. **CloudWatch Logs: no retention policy.** Fix: Set retention on all log groups.
4. **ECR images: no lifecycle policy.** Fix: Add a lifecycle policy to cap old images.
5. **EBS volumes attached to stopped EC2 instances.** Fix: Use `aws ec2 describe-volumes --filters Name=status,Values=available` to find unattached volumes.
6. **RDS running 24/7 in dev.** Fix: Stop dev RDS instances on a schedule (RDS stops automatically resume after 7 days, but CloudWatch Events can restart the stop cycle).

**Q: How do I get a cost breakdown by resource/tag without surprises?**

1. **Enable Cost Allocation Tags:** Billing → Cost Allocation Tags → Activate tags (`Environment`, `Team`, `Project`). Apply these tags to all resources.
2. **AWS Cost Explorer:** View costs by service, region, and tag. Set up daily anomaly detection.
3. **AWS Budgets:** Create a budget alert — email when monthly forecast exceeds $X.

---

## 🏛️ Observability

**Q: How do I see the logs of a running ECS Fargate task without SSH?**

```bash
# Tail logs in real-time from the CLI
aws logs tail /ecs/myapp --follow --format short

# Or: filter for ERROR-level logs in the last hour
aws logs filter-log-events \
  --log-group-name /ecs/myapp \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --filter-pattern "ERROR"
```

**Q: My CloudWatch alarm is in `INSUFFICIENT_DATA` state. Why?**

`INSUFFICIENT_DATA` means the metric has no data points in the evaluation window. Common causes:

1. The resource doesn't exist yet or was deleted (alarm is orphaned).
2. The metric namespace or dimension names are wrong (typo in `ClusterName`, `ServiceName`).
3. The service hasn't emitted any metrics yet (ECS Container Insights might not be enabled).
4. The evaluation period is too short — there are 0 data points to evaluate.
