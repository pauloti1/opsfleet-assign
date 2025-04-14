# 🚀 Karpenter on Amazon EKS with Terraform & Helm

This project provisions an Amazon EKS cluster using Terraform and installs [Karpenter](https://karpenter.sh) using the Helm provider. It also configures a NodePool that supports **Spot Instances** and both **amd64 (x86_64)** and **arm64** architectures.

---

## 📦 Features

- 🧱 EKS cluster provisioned with `terraform-aws-eks`
- 📦 Helm provider installs Karpenter automatically
- ☁️ Spot instance support for cost optimization
- 🧬 Supports both `amd64` and `arm64` architectures
- 🧠 Example EC2NodeClass and NodePool manifests
- 📛 Tagged subnets for Karpenter auto-discovery

---

## 🧰 Prerequisites

- Terraform ≥ 1.3+
- AWS CLI configured (`aws configure`)
- `kubectl` installed
- `helm` (optional but useful for validation)
- IAM credentials with suffiecient permissions 

---

## 🧱 Infrastructure Overview

### 🔹 VPC Configuration

- **CIDR Block:** `10.0.0.0/16`
- **Subnets:**
  - Private subnets for EKS worker nodes
  - Public subnets for ELB
  - Intra subnets for EKS control plane

### 🔹 EKS Cluster

- **Cluster Name:** `ex-<directory_name>`
- **Kubernetes Version:** `1.31`
- **Add-ons Enabled:**
  - CoreDNS
  - Kube-proxy
  - VPC CNI
  - EKS Pod Identity Agent
- **Access:**
  - Public endpoint enabled
  - Creator admin permissions enabled

### 🔹 Node Group (Karpenter Managed)

- **AMI Type:** `BOTTLEROCKET_x86_64`
- **Instance Type:** `t3.large`
- **Size:** 2 desired, 2 min, 3 max
- **Labeled for Karpenter Controller** (`karpenter.sh/controller=true`)

### 🔹 Karpenter

- Installed via Helm chart (`1.1.1`) from ECR Public OCI
- Supports EC2NodeClass provisioning
- Uses queue name from `karpenter` module
- Includes additional IAM policy for SSM

---

## 📦 Modules Used

- `terraform-aws-modules/vpc/aws`
- `terraform-aws-modules/eks/aws`
- `terraform-aws-modules/eks/aws//modules/karpenter`

---

## 🔧 Providers

- `aws`
- `helm`
- `kubernetes`

---

## 🚀 How to Deploy

1. **Initialize Terraform Configuration and deploy resources**

   ```bash
   terraform init
   terraform apply


## 📄 Karpenter NodePool YAML

After applying the Terraform configuration, you need to create a default NodePool for Karpenter. Use the following manifest:

kubectl apply -f nodepool.yaml -n kube-system

Save the content below into a file named `nodepool.yaml`:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["c", "m", "r"]
        - key: "karpenter.k8s.aws/instance-cpu"
          operator: In
          values: ["4", "8", "16", "32"]
        - key: "karpenter.k8s.aws/instance-hypervisor"
          operator: In
          values: ["nitro"]
        - key: "karpenter.k8s.aws/instance-generation"
          operator: Gt
          values: ["2"]
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["spot"]
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
