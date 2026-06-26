# AWS Client VPN Private Access Lab README

## Table of Contents

- [Goal](#goal)
- [Prerequisites](#1-prerequisites)
- [Create certificates](#2-create-certificates)
  - [Create the CA](#create-the-ca)
  - [Create the server certificate](#create-the-server-certificate)
  - [Create the client certificate](#create-the-client-certificate)
- [Import certificates to ACM](#3-import-certificates-to-acm)
- [Create the Client VPN endpoint](#4-create-the-client-vpn-endpoint)
- [Associate a subnet](#5-associate-a-subnet)
- [Security Group](#6-security-group)
- [Authorization rule](#7-authorization-rule)
- [Route table](#8-route-table)
- [Download client config](#9-download-client-config)
- [Embed client cert and key in the OVPN file](#10-embed-client-cert-and-key-in-the-ovpn-file)
- [MTU fix (optional)](#11-mtu-fix-optional)
- [Windows connect](#12-windows-connect)
- [Linux connect](#13-linux-connect)
- [Test private EC2 access](#14-test-private-ec2-access)
- [Test Kubernetes API access](#15-test-kubernetes-api-access)
- [Notes and missing requirements](#16-notes-and-missing-requirements)
- [Troubleshooting](#16-troubleshooting)
- [Production flow](#production-flow)

## Goal

Access private AWS resources (EC2, Kubernetes API server, or private EKS endpoint) from a laptop using AWS Client VPN.

Architecture:

Laptop | AWS Client VPN | VPC Private Subnet | EC2 / Kubernetes Control Plane

## 1. Prerequisites

Required software:

Windows:
- AWS VPN Client or OpenVPN
- OpenSSL
- AWS CLI
- kubectl

Linux:
- openvpn
- openssl
- awscli
- kubectl

Other requirements:
- AWS account with permissions to create ACM certificates, VPC resources, and Client VPN endpoints.
- VPC with a private subnet and route table.
- Local AWS CLI configured with `aws configure` if using AWS CLI commands.

## 2. Create certificates

### Create the CA

Copy/paste on Windows or Linux:

```bash
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 365 \
  -key ca.key \
  -out ca.crt
```

Enter when prompted:
- Country Name: `IN`
- Common Name: `VPN-CA`

### Create the server certificate

Generate server key:

```bash
openssl genrsa -out server.key 2048
```

Create server CSR:

```bash
openssl req -new \
  -key server.key \
  -out server.csr
```

Sign the server certificate:

```bash
openssl x509 -req \
  -in server.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server.crt \
  -days 365
```

### Create the client certificate

Generate client key:

```bash
openssl genrsa -out client.key 2048
```

Create client CSR:

```bash
openssl req -new \
  -key client.key \
  -out client.csr
```

Sign the client certificate:

```bash
openssl x509 -req \
  -in client.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out client.crt \
  -days 365
```

Generated files:

```text
ca.crt ca.key
server.crt server.key
client.crt client.key
```

## 3. Import certificates to ACM

Import the server certificate into ACM using:

```text
Server certificate: server.crt
Private key: server.key
Certificate chain: ca.crt
```

## 4. Create the Client VPN endpoint

In the AWS Console:
1. Open VPC.
2. Select Client VPN Endpoints.
3. Click Create Client VPN Endpoint.

Recommended settings:
- Authentication: Certificate based
- Server certificate: ACM server certificate
- Client CIDR: `10.1.0.0/19`
- IP address type: IPv4

## 5. Associate a subnet

In Client VPN > Target network associations, associate at least one private subnet from your VPC.

## 6. Security Group

Allow Kubernetes API access from the Client VPN CIDR.

Example security group rule:

```text
Type:        Custom TCP
Protocol:    TCP
Port range:  6443
Source:      10.1.0.0/19
```

## 7. Authorization rule

In Client VPN > Authorization rules, add a rule for the VPC CIDR:

```text
Destination network: 172.31.0.0/16
Grant access to:     All users
```

## 8. Route table

Add a route to the Client VPN route table:

```text
Destination: 172.31.0.0/16
Target:      Associated subnet
```

## 9. Download client config

From the Client VPN endpoint page, download the client configuration file. Expected filename:

```text
client-config.ovpn
```

## 10. Embed client cert and key in the OVPN file

Open `client-config.ovpn` and add the client certificate and key blocks:

```text
<cert>
-----BEGIN CERTIFICATE-----
... paste contents of client.crt ...
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
... paste contents of client.key ...
-----END PRIVATE KEY-----
</key>
```

## 11. MTU fix (optional)

If `kubectl` TLS commands hang, add these lines to `client-config.ovpn`:

```text
tun-mtu 1200
mssfix 1160
```

## 12. Windows connect

Import the `.ovpn` file into AWS VPN Client or OpenVPN, then connect.

Verify the VPN adapter:

```powershell
ipconfig
```

You should see a new VPN interface.

## 13. Linux connect

Install OpenVPN if needed:

```bash
sudo apt install openvpn
```

Start the VPN:

```bash
sudo openvpn --config client-config.ovpn
```

## 14. Test private EC2 access

Ping an EC2 private IP:

```powershell
ping 172.31.x.x
```

On Windows, test SSH port access:

```powershell
Test-NetConnection 172.31.x.x -Port 22
```

On Linux, test with netcat if installed:

```bash
nc -vz 172.31.x.x 22
```

## 15. Test Kubernetes API access

Check kubeconfig:

```bash
kubectl config view
```

The cluster server URL should be the private endpoint, for example:

```text
https://172.31.x.x:6443
```

On Windows, test API connectivity:

```powershell
Test-NetConnection 172.31.x.x -Port 6443
```

On Linux, test with kubectl:

```bash
kubectl get nodes
```

## 16. Notes and missing requirements

If you do not already have a VPC, subnet, or route table, create them before starting.

Ensure:
- The VPC CIDR and Client VPN CIDR do not overlap.
- The security group allows port 6443 from the VPN range.
- AWS IAM permissions are available for ACM, VPC, and Client VPN.
- `aws configure` is set up if using the AWS CLI.

curl.exe -k https://172.31.x.x:6443/version
```

# 16. Troubleshooting

TCP works but kubectl timeout:

Check MTU:

``` powershell
ping 172.31.x.x -f -l 1400
```

Reduce:

``` powershell
ping 172.31.x.x -f -l 1200
```

If 1200 works:

Add:

``` text
tun-mtu 1200
mssfix 1160
```

# Production flow

Developer

| 

VPN / SSO

| 

Private Kubernetes endpoint

| 

IAM authentication

| 

RBAC authorization

| 

Namespace access
