# AWS EKS Infrastructure with Terraform

This Terraform configuration creates a complete Amazon EKS (Elastic Kubernetes Service) cluster with supporting AWS infrastructure, including VPC networking, security groups, IAM roles, and Kubernetes add-ons.

## 🚀 What This Infrastructure Creates

### AWS Resources Created

#### **Networking & VPC**
- **VPC**: Main VPC with CIDR `172.20.0.0/16`
- **Internet Gateway**: For public internet access
- **NAT Gateway**: Single NAT Gateway with Elastic IP for private subnet outbound traffic
- **Subnets**:
  - **Public Subnets**: `172.20.64.0/19` (us-west-2a), `172.20.96.0/19` (us-west-2b)
  - **Private Subnets**: `172.20.0.0/19` (us-west-2a), `172.20.32.0/19` (us-west-2b)
- **Route Tables**: Separate routing for public and private subnets

#### **EKS Cluster**
- **EKS Cluster**: Version 1.30 with private and public endpoint access
- **Node Group**: Single node group with t3.large instances (Bottlerocket AMI)
- **Auto-scaling**: Configured for 1-1 nodes (can be scaled as needed)

#### **Storage**
- **EFS File System**: Elastic File System for persistent storage
- **EFS Mount Targets**: Mounted in both public subnets
- **Storage Classes**: EBS (gp3) and EFS storage classes for Kubernetes

#### **Security & Identity**
- **IAM Roles**: EKS cluster role, node group role, EBS/EFS CSI driver roles
- **OpenID Connect Provider**: For IAM roles for service accounts (IRSA)
- **Pod Identity**: EKS Pod Identity Agent for modern IAM authentication
- **Developer User**: IAM user with EKS read-only access

#### **Kubernetes Add-ons**
- **CoreDNS**: DNS service for the cluster
- **Kube Proxy**: Network proxy for Kubernetes services
- **VPC CNI**: Container Network Interface for pod networking
- **EBS CSI Driver**: Container Storage Interface for EBS volumes
- **EFS CSI Driver**: Container Storage Interface for EFS volumes
- **Pod Identity Agent**: For secure pod-to-AWS service authentication
- **Metrics Server**: Resource metrics collection for HPA

## 🏗️ AWS Services Used

| Service | Purpose |
|---------|---------|
| **VPC** | Virtual Private Cloud with subnets and routing |
| **EC2** | EKS worker nodes (t3.large instances) |
| **EKS** | Managed Kubernetes control plane |
| **EFS** | Elastic File System for shared persistent storage |
| **EBS** | Elastic Block Store for pod persistent volumes |
| **IAM** | Identity and Access Management for roles and policies |
| **ELB** | Elastic Load Balancing (NLB for ingress) |
| **NAT Gateway** | Network Address Translation for private subnet internet access |
| **Internet Gateway** | Internet access for public subnets |
| **Route 53** | DNS resolution (via VPC DNS) |
| **CloudWatch** | Monitoring and logging |

## 🏛️ Architecture Overview

### VPC Network Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────────┐
│          Internet Gateway           │
│          (Public Traffic)           │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│         NAT Gateway                 │
│    (Private Subnet Outbound)        │
└─────────────────┬───────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
┌───────────────┐   ┌───────────────┐
│ Public Subnet │   │ Public Subnet │
│ us-west-2a    │   │ us-west-2b    │
│ 172.20.64.0/19│   │172.20.96.0/19│
│               │   │               │
│ • Load Balancers│  │ • EFS Mount  │
│ • NAT Gateway  │  │   Targets     │
└───────────────┘   └───────────────┘
        │                   │
        └─────────┬─────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│         Private Subnets             │
│       (EKS Worker Nodes)            │
├─────────────────────────────────────┤
│ Private Zone 1: 172.20.0.0/19      │
│ Private Zone 2: 172.20.32.0/19     │
│                                     │
│ • EKS Node Group (t3.large)         │
│ • Kubernetes Pods                   │
│ • Internal Load Balancers           │
└─────────────────────────────────────┘
```

### Kubernetes Networking Flow

#### **Ingress Traffic Flow**
```
Internet → Load Balancer (NLB) → Ingress Controller (nginx) → Kubernetes Service → Pod
```

#### **Egress Traffic Flow**
```
Pod → NAT Gateway → Internet
```

#### **Internal Traffic Flow**
```
Pod ↔ Kubernetes Service ↔ Pod (via VPC CNI)
```

### Detailed Network Flow Diagram

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   External User │     │   Load Balancer │     │   Ingress NGINX │
│                 │     │     (NLB)       │     │   Controller     │
│   https://app.com│◄────┤                 │◄────┤                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         ▲                       ▲                       ▲
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Kubernetes     │     │  Kubernetes     │     │      Pod        │
│   Service       │◄────┤   Deployment    │◄────┤   (Application)  │
│   (ClusterIP)   │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                         │
                                                         │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   VPC CNI       │     │   NAT Gateway   │     │   Internet      │
│   (Pod Network) │────►│   (Outbound)    │────►│   Gateway       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## 📋 Prerequisites

Before deploying this infrastructure, ensure you have the following:

### **Required Software**
- **Terraform**: `>= 1.0` (tested with 1.0+)
- **AWS CLI**: Version 2.x with configured credentials
- **kubectl**: For Kubernetes cluster management (optional)
- **Helm**: Version 3.x (optional, for manual deployments)

### **AWS Requirements**
- **AWS Account**: With sufficient permissions to create all resources
- **IAM User/Role**: With the following permissions:
  ```
  - EC2 full access
  - EKS full access
  - IAM full access
  - VPC full access
  - EFS full access
  - CloudWatch full access
  ```

### **Local Environment Setup**
1. **Install AWS CLI v2**:
   ```bash
   # macOS
   brew install awscli

   # Ubuntu/Debian
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   ```

2. **Configure AWS Credentials**:
   ```bash
   aws configure
   # Enter your AWS Access Key ID, Secret Access Key, and default region (us-west-2)
   ```

3. **Install Terraform**:
   ```bash
   # macOS
   brew install terraform

   # Ubuntu/Debian
   wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install terraform
   ```

## 🚀 Installation & Deployment

### **Step 1: Clone and Navigate**
```bash
git clone <repository-url>
cd eks-terraform-bkp
```

### **Step 2: Initialize Terraform**
```bash
terraform init
```
This command:
- Downloads required providers (AWS ~5.53)
- Initializes the working directory
- Creates `.terraform` directory with provider plugins

### **Step 3: Review the Plan**
```bash
terraform plan
```
This command:
- Analyzes the configuration
- Shows what resources will be created, modified, or destroyed
- Validates syntax and dependencies
- **Important**: Review the output carefully before applying

### **Step 4: Deploy the Infrastructure**
```bash
terraform apply
```
This command:
- Creates all AWS resources defined in the configuration
- May take 15-25 minutes to complete
- Prompts for confirmation (add `--auto-approve` for automation)

**Expected Output**:
```
Apply complete! Resources: 45 added, 0 changed, 0 destroyed.
```

### **Step 5: Configure kubectl (Optional)**
After deployment, configure kubectl to access your cluster:
```bash
aws eks update-kubeconfig --region us-west-2 --name test-terraform-test-terraform
kubectl get nodes
```

## 🗑️ Cleanup & Destruction

### **Destroy All Resources**
```bash
terraform destroy
```
This command:
- Destroys all resources created by this configuration
- May take 10-15 minutes to complete
- Prompts for confirmation (add `--auto-approve` for automation)

**⚠️ Warning**: This will permanently delete all resources including:
- EKS cluster and all workloads
- EFS file systems and data
- VPC, subnets, and networking components
- IAM roles and policies

### **Selective Resource Destruction**
If you need to destroy specific resources, comment them out in the `.tf` files and run:
```bash
terraform plan -destroy
terraform apply -destroy
```

## 🔍 Post-Deployment Verification

### **Check AWS Resources**
```bash
# Check EKS cluster
aws eks list-clusters --region us-west-2

# Check EC2 instances (worker nodes)
aws ec2 describe-instances --region us-west-2 --filters "Name=tag:Name,Values=*test-terraform*" --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]'

# Check VPC and subnets
aws ec2 describe-vpcs --region us-west-2 --filters "Name=tag:Name,Values=*test-terraform*"
```

### **Check Kubernetes Resources**
```bash
# Get cluster info
kubectl cluster-info

# Check nodes
kubectl get nodes -o wide

# Check pods (system components)
kubectl get pods -A

# Check storage classes
kubectl get storageclass

# Check services
kubectl get svc -A
```

### **Test Storage**
```bash
# Create a test PVC using EBS
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 1Gi
EOF

# Check PVC status
kubectl get pvc
```

## 🛠️ Customization

### **Scaling the Cluster**
Edit `8-nodes.tf` to modify node group scaling:
```hcl
scaling_config {
  desired_size = 2  # Change from 1 to 2
  max_size     = 5  # Allow scaling up to 5 nodes
  min_size     = 1
}
```

### **Adding Node Groups**
Add additional node groups in `8-nodes.tf` for different workloads.

### **Modifying Instance Types**
Change instance types in the `instance_types` list in `8-nodes.tf`.

### **Adding Ingress Controller**
Deploy NGINX Ingress Controller:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx -f values/nginx-ingress.yaml
```

## 🔒 Security Considerations

- **Network Security**: All worker nodes are in private subnets
- **IAM**: Least-privilege access with specific IAM roles
- **Encryption**: EFS is configured but not encrypted (modify for production)
- **Pod Security**: Consider implementing Pod Security Standards
- **Network Policies**: Implement Kubernetes Network Policies for pod-to-pod traffic

## 📊 Cost Estimation

**Approximate Monthly Costs** (us-west-2):
- **EKS Cluster**: ~$73/month
- **EC2 (t3.large)**: ~$62/month (1 instance)
- **EFS**: ~$0-3/month (depending on usage)
- **NAT Gateway**: ~$33/month
- **Data Transfer**: Variable based on traffic
- **Total Estimate**: ~$170-200/month (minimum)

## 🆘 Troubleshooting

### **Common Issues**

1. **Terraform Apply Fails**
   - Check AWS credentials: `aws sts get-caller-identity`
   - Verify region: `aws configure list`
   - Check service limits/quota

2. **EKS Cluster Creation Timeout**
   - Wait longer (can take 15-20 minutes)
   - Check CloudTrail for errors
   - Verify VPC/subnet configuration

3. **Node Group Creation Fails**
   - Check IAM permissions for node group role
   - Verify subnet configuration
   - Check security groups

4. **Helm Deployments Fail**
   - Ensure cluster is fully ready: `aws eks describe-cluster --name <cluster-name>`
   - Check OIDC provider setup
   - Verify IAM roles for service accounts

### **Logs and Debugging**
```bash
# Terraform logs
export TF_LOG=DEBUG
terraform apply

# AWS CloudTrail events
aws cloudtrail lookup-events --region us-west-2 --max-items 10

# EKS cluster logs
aws eks describe-cluster --name test-terraform-test-terraform --region us-west-2
```

## 📚 Additional Resources

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes with proper documentation
4. Test thoroughly
5. Submit a pull request

---

**Note**: This configuration is designed for development/testing environments. For production use, consider:
- Enabling encryption for EFS and EBS
- Implementing backup strategies
- Adding monitoring and alerting
- Configuring proper security groups
- Implementing multi-AZ redundancy for critical components
