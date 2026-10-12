# VProfile Infrastructure Repository

This repository contains the Terraform configuration that creates the AWS foundation for the VProfile GitOps platform. It defines the network, Amazon EKS control plane and worker nodes, the IAM roles required by EKS, and the Amazon EBS CSI driver integration.

Infrastructure as Code (IaC) keeps infrastructure changes version-controlled, reviewable, repeatable, and auditable. The configuration in this repository represents the desired state; Terraform compares that state with AWS during planning and applies the approved changes.

The broader platform is illustrated as a DevOps / DevSecOps / GitOps workflow. This repository owns only the infrastructure pieces defined in its Terraform files. The application build pipeline and Helm/Argo CD desired-state configuration belong in separate repositories.

## Architecture overview

![VProfile GitOps DevOps architecture](assets/gitops-architecture.png)

The diagram describes the intended end-to-end platform: an application pipeline builds and scans a container image, a Helm repository expresses the desired Kubernetes state, and Argo CD synchronizes that state to EKS. Terraform provides the EKS foundation on which that workflow can run.

### What this repository provisions

- A VPC with DNS support enabled.
- An internet gateway, one public route table, and public subnets across the configured Availability Zones.
- An EKS control plane and a managed EKS node group.
- IAM roles and AWS-managed policy attachments for the EKS control plane and worker nodes.
- An EKS OIDC provider and an IAM role for service-account-based access (IRSA) by the Amazon EBS CSI driver.
- The `aws-ebs-csi-driver` EKS add-on.

### Platform components outside this Terraform configuration

The architecture diagram is wider in scope than this repository. The following are **not** created by the Terraform code currently present:

| Component                               | Implementation and Responsibility                                                                                                                                                        |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GitHub and Version Control              | The repository is hosted on GitHub and contains the version-controlled Terraform configuration for AWS infrastructure.                                                                   |
| GitHub Actions                          | CI/CD workflows for Terraform validation, planning, deployment, and drift detection are not configured in this repository.                                                               |
| Application Build and Testing           | Maven build, unit tests, Checkstyle, and SonarQube analysis belong to the application CI pipeline.                                                                                       |
| Containerization and Security Scanning  | Docker image builds and Trivy vulnerability scanning belong to the application CI pipeline.                                                                                              |
| Amazon ECR                              | Container image registry management belongs to the application delivery infrastructure and is not defined in this Terraform configuration.                                               |
| Amazon EKS                              | Terraform provisions the EKS control plane and managed worker node group.                                                                                                                |
| IAM and EBS CSI Driver                  | Terraform configures EKS IAM roles, the OIDC provider, IRSA for the EBS CSI driver, and the EBS CSI EKS add-on.                                                                          |
| Argo CD                                 | `argocd-ingress.yaml` defines an ingress for an existing Argo CD installation. Installing Argo CD and managing its Application resources are separate tasks.                             |
| AWS Load Balancer Controller            | The controller is not installed by Terraform. `iam_policy.json` provides a policy document for a separately configured controller.                                                       |
| ACM, DNS, and Application Load Balancer | The Argo CD ingress references an existing ACM certificate and hostname. Certificate issuance, DNS records, and load balancer lifecycle are not managed by this Terraform configuration. |
| Amazon CloudWatch                       | CloudWatch logging, metrics, dashboards, and alarms are not configured in the current Terraform code.                                                                                    |
| SonarQube on EC2                        | The EC2 instance and SonarQube server infrastructure are not defined in this repository.                                                                                                 |
| Terraform State Management              | Terraform uses an S3 remote backend configured in `backend.tf`. The backend bucket must exist before initialization.                                                                     |
| Terraform Drift Detection               | Drift can be inspected manually using `terraform plan`. Scheduled detection and automated remediation are not configured.                                                                |
| Slack Notifications                     | Slack integration for deployment results, Terraform drift, or infrastructure alerts is not configured.                                                                                   |


## Repository responsibilities

The configured Git remote identifies this repository as [`7azeeem/vprofile-infra`](https://github.com/7azeeem/vprofile-infra). No Application or Helm repository URLs are available in this workspace, so they are intentionally not guessed below.

| Repository | Responsibility | Main technologies |
| --- | --- | --- |
| `vprofile-infra` (this repository) | Provision AWS networking, EKS, IAM integration, and the EBS CSI add-on. | Terraform, AWS, Amazon EKS, IAM, EBS CSI |
| Application repository (external) | Build, test, analyze, scan, and publish the VProfile application image. | Application tooling, CI, container tooling |
| Helm repository (external) | Define Kubernetes desired state and Argo CD application configuration. | Helm, Kubernetes, Argo CD |

## Infrastructure components

### Network

Terraform creates one VPC using `vpc_cidr`, an internet gateway, a public route table with a default route to that gateway, and one public subnet for each value in `public_subnets`. The corresponding `availability_zones` entry is used for each subnet.

Each subnet assigns public IP addresses on launch and is tagged for external Kubernetes load balancers and cluster discovery. There are no private subnets, NAT gateways, network ACLs, custom security groups, or VPC endpoints in this configuration. Treat this as a public-subnet EKS design; do not describe it as a private network topology.

### Amazon EKS and worker nodes

The EKS cluster uses all created public subnets. The managed node group also uses those subnets and applies the configured `instance_type`, `min_size`, `max_size`, and `desired_size` values. By default, that is one `t3.large` worker node, with scaling permitted from one to two nodes.

The configuration does not set EKS endpoint access, Kubernetes version, logging, encryption configuration, node labels, taints, or a custom launch template. AWS and provider defaults therefore apply where applicable.

### IAM roles

The cluster role receives `AmazonEKSClusterPolicy`. The worker-node role receives:

- `AmazonEKSWorkerNodePolicy`
- `AmazonEKS_CNI_Policy`
- `AmazonEC2ContainerRegistryReadOnly`

These are AWS-managed policies necessary for normal EKS control-plane and node operation. They are not a substitute for a review of the permissions granted to CI users, human operators, or Kubernetes workloads.

### OIDC, IRSA, and Amazon EBS CSI

Terraform derives the EKS cluster's OIDC issuer certificate, creates an IAM OIDC provider, and limits the EBS CSI IAM role trust policy to the `kube-system/ebs-csi-controller-sa` service account and `sts.amazonaws.com` audience. The role receives `AmazonEBSCSIDriverPolicy` and is passed to the `aws-ebs-csi-driver` add-on.

This enables the EBS CSI controller to request AWS credentials through IRSA rather than placing the storage policy on every worker node. It does not create persistent volumes, storage classes, or application claims; those are Kubernetes/Helm concerns.

### Argo CD ingress manifest

`argocd-ingress.yaml` is a Kubernetes manifest, not a Terraform resource. It defines an internet-facing ALB-style ingress in the `argocd` namespace for a configured host and references an existing ACM certificate. Applying it requires a running cluster, Argo CD, and a compatible AWS Load Balancer Controller that has been installed and granted the required permissions separately.

The included `iam_policy.json` is an IAM policy document for that controller workflow. It is not attached to any role by this Terraform configuration.

## Terraform and infrastructure lifecycle

Terraform uses the AWS provider constrained to `~> 5.0`. The repository configures an S3 remote backend with the state key `eks/terraform.tfstate` in `us-east-1`. The backend bucket must already exist and must be accessible to the identity running Terraform. No DynamoDB locking table is configured, so state locking is not supplied by this repository's backend block.

> The `s3_bucket` input variable is currently unused. The active backend bucket is defined directly in `backend.tf`; normal Terraform input variables cannot parameterize a backend block.

The normal lifecycle is:

1. Edit Terraform configuration and run formatting/validation checks.
2. Initialize the configured provider and S3 backend.
3. Produce and review a plan that shows the delta between configuration, state, and AWS.
4. Apply only the reviewed plan.
5. Use a refresh-enabled plan to identify infrastructure drift; Terraform does not automatically repair drift unless an approved apply is run.

No GitHub Actions workflow, scheduled drift-detection job, Slack notification, approval gate, or automated destroy workflow is committed in this repository. Those stages shown in the diagram are planned platform capabilities, not documented implementation.

## GitHub Actions workflow

This repository has a GitHub remote but contains no files under `.github/workflows/`. Consequently, there are no repository-defined triggers, AWS authentication mechanism, permissions block, approval requirement, plan/apply behavior, schedule, or notification behavior to document.

If automation is added later, it should use narrowly scoped GitHub permissions and short-lived AWS credentials (for example, OIDC), protect the Terraform state, expose the plan for review, and require an explicit approval boundary before production applies. These are recommendations, not a description of the current implementation.

## Prerequisites

Before deploying, ensure that you have:

- An AWS account and an authenticated AWS CLI profile or equivalent AWS credentials with permission to create and manage the resources in the Terraform configuration.
- Access to the pre-existing S3 backend bucket configured in `backend.tf`.
- Terraform compatible with AWS provider `~> 5.0`.
- Git.
- The AWS CLI for authentication and post-deployment EKS access.
- `kubectl` only if you will administer the cluster or apply `argocd-ingress.yaml`.

Review AWS service quotas before deployment, particularly EKS, EC2 On-Demand capacity, VPC resources, IAM, and Elastic Load Balancing quotas. EKS control-plane charges, EC2 worker-node charges, EBS volumes, data transfer, public IPs, and any separately created load balancers can incur costs.

## Configuration

All declared inputs have defaults. You may override them with a `terraform.tfvars` file, a `*.auto.tfvars` file, `-var` arguments, or `TF_VAR_` environment variables. Do not commit secrets or local credential files.

| Variable | Default | Purpose |
| --- | --- | --- |
| `region` | `us-east-1` | AWS provider region. |
| `cluster_name` | `vprofile-eks-cluster` | EKS cluster name and name prefix for several resources. |
| `vpc_cidr` | `10.0.0.0/16` | VPC IPv4 CIDR block. |
| `public_subnets` | `10.0.1.0/24`, `10.0.2.0/24` | Public subnet CIDRs. |
| `availability_zones` | `us-east-1a`, `us-east-1b` | Availability Zone for each public subnet. |
| `instance_type` | `t3.large` | EKS worker node instance type. |
| `min_size` | `1` | Minimum node-group size. |
| `max_size` | `2` | Maximum node-group size. |
| `desired_size` | `1` | Initial desired node-group size. |
| `s3_bucket` | `vprofile-eks-terraform-state` | Declared but currently unused by the active backend. |

`public_subnets` and `availability_zones` must contain compatible, corresponding entries. Choose CIDRs that do not overlap existing networks and Availability Zones that exist in the selected region.

Example non-secret overrides:

```hcl
# terraform.tfvars
region             = "us-east-1"
cluster_name       = "vprofile-eks-cluster"
vpc_cidr           = "10.0.0.0/16"
public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
availability_zones = ["us-east-1a", "us-east-1b"]
instance_type      = "t3.large"
min_size           = 1
max_size           = 2
desired_size       = 1
```

## Deployment

### 1. Authenticate to AWS

Configure AWS authentication using your organization-approved method and verify the target account before changing infrastructure:

```bash
aws sts get-caller-identity
```

### 2. Initialize and validate

The following commands are intended to be run from the repository root:

```bash
terraform fmt -check
terraform init
terraform validate
```

`terraform init` connects to the S3 backend specified in `backend.tf`. If the bucket, region, or access policy is not available, initialization will fail before planning.

### 3. Plan and review

```bash
terraform plan -out=tfplan
```

Review the plan carefully. It should only contain expected additions, changes, or removals for the AWS account and region you selected.

### 4. Apply an approved plan

```bash
terraform apply tfplan
```

Terraform exposes these outputs after a successful apply:

- `cluster_endpoint`
- `cluster_name`
- `cluster_arn`

To configure `kubectl` for the created cluster, use the resulting cluster name and chosen region:

```bash
aws eks update-kubeconfig --region us-east-1 --name vprofile-eks-cluster
kubectl get nodes
```

If you override `region` or `cluster_name`, substitute those values in the command.

### 5. Optional Argo CD ingress

Only after separately installing Argo CD and the AWS Load Balancer Controller, and after reviewing the existing DNS and ACM certificate settings in the manifest, apply the ingress:

```bash
kubectl apply -f argocd-ingress.yaml
```

This manifest is not part of `terraform apply` or `terraform destroy`.

## Drift detection and notifications

A regular Terraform plan refreshes Terraform's view of managed resources and reports differences between configuration, recorded state, and the AWS APIs. You can inspect drift with:

```bash
terraform plan
```

This repository has no scheduled drift-detection workflow and no Slack integration. A detected difference is only a report until someone reviews it and runs an approved `terraform apply`; automatic remediation is not configured.

## Security considerations

### Implemented

- The EBS CSI controller uses a role trust policy limited to its Kubernetes service account and the STS audience.
- EKS cluster and node roles use specific AWS-managed policy attachments rather than administrator access in the Terraform code.
- Terraform state is configured for an S3 backend rather than local state. The bucket's encryption, versioning, access policy, and lifecycle rules are outside this repository and must be verified separately.
- The Argo CD ingress manifest requests HTTPS and redirects HTTP to HTTPS, but its certificate and DNS lifecycle are external to Terraform.

### Operational requirements and recommendations

- Use least-privilege AWS identities for Terraform and keep AWS credentials out of source control.
- Restrict access to the state bucket because state can contain infrastructure metadata and potentially sensitive values.
- Review the public subnet design and the internet-facing Argo CD ingress before using it in a sensitive environment.
- Review and rotate the referenced external certificate and DNS configuration through their owning processes.
- Do not assume security groups, private endpoints, cluster logging, secret management, encryption policies, or CI permissions are configured here; they are not defined in the present Terraform files.

## Cost management and cleanup

This environment can generate AWS charges for the EKS control plane, EC2 worker nodes, EBS storage, load balancers created by Kubernetes integrations, public IPv4 addresses, and data transfer. Destroy only after confirming that the target state, AWS account, and region are correct and that no workloads or persistent data need to be retained.

To review cleanup impact:

```bash
terraform plan -destroy
```

To remove Terraform-managed infrastructure after approval:

```bash
terraform destroy
```

`terraform destroy` can remove the EKS cluster, worker nodes, VPC resources, IAM roles created by this configuration, and the EBS CSI add-on. It does not remove externally managed resources such as the backend S3 bucket or resources created outside this Terraform state. Review the destroy plan before confirming it.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `terraform init` cannot access the backend | Confirm the S3 bucket in `backend.tf` exists, is in `us-east-1`, and your AWS identity has the needed bucket and object permissions. |
| `terraform validate` reports provider issues | Run `terraform init` first and use a Terraform version compatible with the configured provider constraint. |
| Cluster or node-group creation fails | Verify AWS credentials, regional service quotas, selected Availability Zones, subnet CIDR validity, and IAM permissions. |
| Worker nodes do not become ready | Inspect EKS node-group health and confirm the node role policy attachments were created successfully. |
| EBS-backed claims cannot provision volumes | Confirm the EBS CSI add-on is active and that its service account remains `kube-system/ebs-csi-controller-sa`. |
| Argo CD ingress does not create an ALB | Confirm Argo CD, the AWS Load Balancer Controller, its IAM permissions, DNS, and referenced ACM certificate are configured outside this repository. |
| A plan shows unexpected changes | Stop before applying, verify the active AWS account and workspace/backend state, and investigate changes in AWS or configuration. |

## Related repositories

- Infrastructure: [`7azeeem/vprofile-infra`](https://github.com/7azeeem/vprofile-infra)
- Helm repository links:  [`7azeeem/vprofile-helm`](https://github.com/7azeeem/vprofile-helm)
