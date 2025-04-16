# Step-by-Step Guide to Deploy MongoDB and MongoExpress on Amazon EKS using Helm

This guide provides detailed instructions to deploy **MongoDB** and **MongoExpress** on an **Amazon EKS** cluster using **Helm**. You will configure MongoDB as a replica set and expose MongoExpress using an NGINX Ingress Controller.

---

## 📋 Prerequisites

Ensure the following tools and configurations are in place:

- **AWS CLI**: Installed and configured  
  👉 [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

- **kubectl**: Installed and configured  
  👉 [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/)

- **Helm**: Installed  
  👉 [Helm Installation Guide](https://helm.sh/docs/intro/install/)

- **EKS Cluster**: Running and accessible  
  👉 [EKS Getting Started Guide](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)

- **IAM Permissions**: Ability to manage EKS, EC2, and related services

- **kubeconfig**: Update config to connect to your EKS cluster:

```bash
aws eks update-kubeconfig --region <region> --name <cluster-name>
