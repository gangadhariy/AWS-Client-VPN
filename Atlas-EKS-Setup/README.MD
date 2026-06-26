# Atlas EKS Complete Setup Guide

This README explains how to create an Amazon EKS cluster from the AWS Console, configure networking, create managed node groups, configure IAM authentication, and implement Kubernetes RBAC access.

All AWS resource names in this guide use the `atlas-` prefix.

## Table of Contents

- [Create VPC Networking](#1-create-vpc-networking)
- [Create Subnets](#2-create-subnets)
  - [Public Subnet 1](#public-subnet-1)
  - [Public Subnet 2](#public-subnet-2)
  - [Private Subnet 1](#private-subnet-1)
  - [Private Subnet 2](#private-subnet-2)
- [Create Internet Gateway](#3-create-internet-gateway)
- [Create NAT Gateway](#4-create-nat-gateway)
- [Configure Route Tables](#5-configure-route-tables)
  - [Public Route Table](#public-route-table)
  - [Private Route Table](#private-route-table)
- [Create EKS Cluster](#6-create-eks-cluster)
  - [Cluster IAM Role](#cluster-iam-role)
  - [Networking Configuration](#networking-configuration)
- [Create Managed Node Group](#7-create-managed-node-group)
  - [Node IAM Role](#node-iam-role)
  - [Node Group Settings](#node-group-settings)
- [Create Admin EC2 Machine](#8-create-admin-ec2-machine)
- [Create IAM Role for Cluster Administration](#9-create-iam-role-for-cluster-administration)
- [Configure EKS Access Entry](#10-configure-eks-access-entry)
- [Configure kubectl](#11-configure-kubectl)
- [Create Namespaces](#12-create-namespaces)
- [Create IAM Users](#13-create-iam-users)
- [Create Developer Access Entries](#14-create-developer-access-entries)
- [Kubernetes RBAC Permissions](#15-kubernetes-rbac-permissions)
- [Example Developer Role](#example-developer-role)
- [User Authentication Test](#16-user-authentication-test)
- [Final Architecture](#final-architecture)

------------------------------------------------------------------------

# 1. Create VPC Networking

## Create VPC

Open AWS Console:

VPC → Your VPCs → Create VPC

Choose:

- Resource type: VPC only
- Name: `atlas-eks-vpc`
- IPv4 CIDR: `10.0.0.0/16`

Create the VPC.

This VPC will contain the EKS control plane communication, worker nodes, load balancers, and application workloads.

------------------------------------------------------------------------

# 2. Create Subnets

Create four subnets inside `atlas-eks-vpc`.

## Public Subnet 1

Name: `atlas-public-subnet-1`

CIDR: `10.0.1.0/24`

Availability Zone: `us-east-1a`

------------------------------------------------------------------------

## Public Subnet 2

Name: `atlas-public-subnet-2`

CIDR: `10.0.2.0/24`

Availability Zone: `us-east-1b`

------------------------------------------------------------------------

## Private Subnet 1

Name: `atlas-private-subnet-1`

CIDR: `10.0.11.0/24`

Availability Zone: `us-east-1a`

------------------------------------------------------------------------

## Private Subnet 2

Name: `atlas-private-subnet-2`

CIDR: `10.0.12.0/24`

Availability Zone: `us-east-1b`

------------------------------------------------------------------------

# 3. Create Internet Gateway

Go to:

VPC → Internet Gateways → Create Internet Gateway

Create: `atlas-eks-igw`

Attach it to: `atlas-eks-vpc`

The Internet Gateway allows resources in public subnets to communicate with the internet.

------------------------------------------------------------------------

# 4. Create NAT Gateway

Go to:

VPC → NAT Gateways → Create NAT Gateway

Name: `atlas-eks-nat`

Select Subnet: `atlas-public-subnet-1`

Allocate a new Elastic IP.

The NAT Gateway allows private EKS worker nodes to download updates and communicate externally without receiving public IP addresses.

------------------------------------------------------------------------

# 5. Configure Route Tables

## Public Route Table

Create: `atlas-public-rt`

Add route:

```text
0.0.0.0/0 -> atlas-eks-igw
```

Associate:

- `atlas-public-subnet-1`
- `atlas-public-subnet-2`

------------------------------------------------------------------------

## Private Route Table

Create: `atlas-private-rt`

Add route:

```text
0.0.0.0/0 -> atlas-eks-nat
```

Associate:

- `atlas-private-subnet-1`
- `atlas-private-subnet-2`

------------------------------------------------------------------------

# 6. Create EKS Cluster

Go to:

EKS → Clusters → Create

Cluster name: `atlas-eks-cluster`

Select Kubernetes version.

------------------------------------------------------------------------

## Cluster IAM Role

Create IAM role: `atlas-eks-cluster-role`

Trusted entity: `AWS service`

Service: `EKS`

Attach managed policy: `AmazonEKSClusterPolicy`

This role allows the EKS control plane to manage AWS resources required for Kubernetes.

------------------------------------------------------------------------

## Networking Configuration

Select:

- VPC: `atlas-eks-vpc`
- Subnets:
  - `atlas-private-subnet-1`
  - `atlas-private-subnet-2`
- Endpoint access: `Public and Private`

Create the cluster.

Wait until the cluster status becomes `ACTIVE`.

------------------------------------------------------------------------

# 7. Create Managed Node Group

Open:

EKS → `atlas-eks-cluster` → Compute → Add Node Group

Node group name: `atlas-managed-node-group`

------------------------------------------------------------------------

## Node IAM Role

Create IAM role: `atlas-eks-node-role`

Attach managed policies:

- `AmazonEKSWorkerNodePolicy`
- `AmazonEKS_CNI_Policy`
- `AmazonEC2ContainerRegistryReadOnly`

This role allows worker nodes to join the cluster, use networking, and pull container images.

------------------------------------------------------------------------

## Node Group Settings

Choose:

- Instance type: `t3.medium`
- Scaling:
  - Minimum: `2`
  - Desired: `2`
  - Maximum: `4`
- Subnets:
  - `atlas-private-subnet-1`
  - `atlas-private-subnet-2`

Create the node group.

------------------------------------------------------------------------

# 8. Create Admin EC2 Machine

Create an Ubuntu EC2 instance:

- Name: `atlas-eks-admin-host`
- Attach the appropriate IAM role or AWS CLI credentials to access the EKS cluster.

Install:

- AWS CLI
- kubectl

Install AWS CLI using the AWS docs:

```text
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
```

Verify:

```bash
aws --version
```

------------------------------------------------------------------------

# 9. Create IAM Role for Cluster Administration

Create IAM role: `atlas-eks-admin-role`

Attach managed policy: `AmazonEKSClusterAdminPolicy`

This role will be used for cluster administrator access.

------------------------------------------------------------------------

# 10. Configure EKS Access Entry

In the EKS console, go to:

EKS → `atlas-eks-cluster` → Access → Add access entry

Choose:

- Principal: `atlas-eks-admin-role`
- Type: `Standard`
- Attach policy: `AmazonEKSClusterAdminPolicy`

This maps the IAM role to Kubernetes administrator permissions for the cluster.

------------------------------------------------------------------------

# 11. Configure kubectl

Run this from the admin host or a machine with AWS credentials:

```bash
aws eks update-kubeconfig --region us-east-1 --name atlas-eks-cluster
```

Verify cluster access:

```bash
kubectl get nodes
```

The administrator should now access the cluster.

------------------------------------------------------------------------

# 12. Create Namespaces

Create application namespaces:

```bash
kubectl create namespace atlas-dev-ns
kubectl create namespace atlas-stage-ns
```

------------------------------------------------------------------------

# 13. Create IAM Users

Create these IAM users in AWS:

- `atlas-dev-user-1`
- `atlas-dev-user-2`
- `atlas-stage-user-1`
- `atlas-stage-user-2`

Create console or programmatic access as needed.

------------------------------------------------------------------------

# 14. Create Developer Access Entries

In the EKS console Access section, add access entries for the developer users.

Developer group:

- Users: `atlas-dev-user-1`, `atlas-dev-user-2`
- Kubernetes group: `atlas-dev-team`

Stage group:

- Users: `atlas-stage-user-1`, `atlas-stage-user-2`
- Kubernetes group: `atlas-stage-team`

Each group should be mapped to an appropriate IAM principal or role.

------------------------------------------------------------------------

# 15. Kubernetes RBAC Permissions

Create namespace roles for `atlas-dev-ns` and `atlas-stage-ns`.

Developers should be able to:

- create resources
- delete resources
- update resources
- list resources
- get resources

Allowed resources:

- Pods
- Deployments
- StatefulSets
- DaemonSets
- Services
- ConfigMaps
- PersistentVolumeClaims

Not allowed:

- Secrets

Example Role YAML for `atlas-dev-ns`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: atlas-dev-role
  namespace: atlas-dev-ns
rules:
- apiGroups: ["", "apps"]
  resources:
  - pods
  - deployments
  - statefulsets
  - daemonsets
  - services
  - configmaps
  - persistentvolumeclaims
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
```

Example RoleBinding YAML:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: atlas-dev-binding
  namespace: atlas-dev-ns
subjects:
- kind: Group
  name: atlas-dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: atlas-dev-role
  apiGroup: rbac.authorization.k8s.io
```

This grants the `atlas-dev-team` group access only inside the `atlas-dev-ns` namespace.

------------------------------------------------------------------------

# 16. User Authentication Test

Login as a developer user and verify identity:

```bash
aws sts get-caller-identity
```

Configure kubectl with the cluster context again if needed:

```bash
aws eks update-kubeconfig --region us-east-1 --name atlas-eks-cluster
```

Test allowed operations:

```bash
kubectl get pods -n atlas-dev-ns
kubectl create deployment nginx --image nginx -n atlas-dev-ns
```

Test blocked operations:

```bash
kubectl get secrets -n atlas-dev-ns
```

This command should return `Forbidden` because secrets are not included in the RBAC permissions.

------------------------------------------------------------------------

# Final Architecture

IAM User / Role → EKS Access Entry → Kubernetes Group → RoleBinding → Namespace Role → Allowed Kubernetes Resources


This role allows worker nodes to join the cluster, use networking, and
pull container images.

------------------------------------------------------------------------

## Node Group Settings

Instance type:

t3.medium

Scaling:

Minimum: 2

Desired: 2

Maximum: 4

Subnets:

-   atlas-private-subnet-1
-   atlas-private-subnet-2

Create the node group.

------------------------------------------------------------------------

# 8. Create Admin EC2 Machine

Create an Ubuntu EC2 instance:

Name:

atlas-eks-admin-host

Install:

-   AWS CLI
-   kubectl

Install AWS CLI using:

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

Verify:

aws --version

------------------------------------------------------------------------

# 9. Create IAM Role for Cluster Administration

Create role:

atlas-eks-admin-role

Attach:

AmazonEKSClusterAdminPolicy

This role will be used for cluster administrator access.

------------------------------------------------------------------------

# 10. Configure EKS Access Entry

Go:

EKS → atlas-eks-cluster → Access → Create Access Entry

Principal:

atlas-eks-admin-role

Type:

Standard

Attach policy:

AmazonEKSClusterAdminPolicy

This maps the IAM role to Kubernetes administrator permissions.

------------------------------------------------------------------------

# 11. Configure kubectl

Run:

aws eks update-kubeconfig\
--region us-east-1\
--name atlas-eks-cluster

Verify:

kubectl get nodes

The administrator should now access the cluster.

------------------------------------------------------------------------

# 12. Create Namespaces

Create application namespaces:

kubectl create namespace atlas-dev-ns

kubectl create namespace atlas-stage-ns

------------------------------------------------------------------------

# 13. Create IAM Users

Create:

-   atlas-dev-user-1
-   atlas-dev-user-2
-   atlas-stage-user-1
-   atlas-stage-user-2

------------------------------------------------------------------------

# 14. Create Developer Access Entries

Create EKS Access Entries:

Developer users:

atlas-dev-user-1 atlas-dev-user-2

Kubernetes group:

atlas-dev-team

Stage users:

atlas-stage-user-1 atlas-stage-user-2

Kubernetes group:

atlas-stage-team

------------------------------------------------------------------------

# 15. Kubernetes RBAC Permissions

Create namespace Roles.

Users should be able to:

-   create resources
-   delete resources
-   update resources
-   list resources
-   get resources

Allowed:

-   Pods
-   Deployments
-   StatefulSets
-   DaemonSets
-   Services
-   ConfigMaps
-   PVC

Not allowed:

-   Secrets

------------------------------------------------------------------------

# Example Developer Role

Create Role:

atlas-dev-role

Namespace:

atlas-dev-ns

Bind:

atlas-dev-team

Create a RoleBinding:

atlas-dev-binding

This gives developers access only inside atlas-dev-ns.

------------------------------------------------------------------------

# 16. User Authentication Test

Login as a developer user.

Verify identity:

aws sts get-caller-identity

Configure kubectl:

aws eks update-kubeconfig\
--region us-east-1\
--name atlas-eks-cluster

Test allowed operations:

kubectl get pods -n atlas-dev-ns

kubectl create deployment nginx\
--image nginx\
-n atlas-dev-ns

Test blocked operations:

kubectl get secrets -n atlas-dev-ns

The command should return Forbidden because secrets are not included in
the RBAC permissions.

------------------------------------------------------------------------

# Final Architecture

IAM User / Role

↓

EKS Access Entry

↓

Kubernetes Group

↓

RoleBinding

↓

Namespace Role

↓

Allowed Kubernetes Resources
