---
title: "S3VpeOverVpn"
date: 2026-04-18T18:06:42+10:00
draft: false
tags: ["AWS", "data transfer"]
categories: ["AWS"]
featured_image: ""
summary: "Using S3 private endpoint over a VPN connection."
---

# S3 Interface Endpoint Upload from OCI Oracle Linux VM over VPN

End-to-end instructions using `/etc/hosts` for DNS and IAM access keys for auth. Assumes the IPsec VPN between your OCI VCN and AWS VPC is up and routing in both directions.

**Example values used throughout** (swap for yours):
- AWS Region: `us-east-1`
- AWS VPC CIDR: `10.0.0.0/16`
- OCI VCN CIDR: `10.200.0.0/16`
- S3 Bucket: `my-bucket`
- OCI VM: Oracle Linux 8 or 9

---

## Step 1: Create Security Group for the S3 Interface Endpoint

In the AWS Console → VPC → Security Groups → Create security group:

- **Name**: `sg-vpce-s3`
- **VPC**: your VPC
- **Inbound rule**: HTTPS (TCP 443) from `10.200.0.0/16` (your OCI VCN CIDR)
- **Outbound**: leave default

CLI equivalent:

```bash
aws ec2 create-security-group \
  --group-name sg-vpce-s3 \
  --description "S3 interface endpoint access from OCI" \
  --vpc-id vpc-xxxxxxxx

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --protocol tcp --port 443 \
  --cidr 10.200.0.0/16
```

---

## Step 2: Create the S3 Interface Endpoint

AWS Console → VPC → Endpoints → Create endpoint:

- **Name**: `vpce-s3-interface`
- **Service category**: AWS services
- **Service**: search `s3`, select the one with **Type: Interface** → `com.amazonaws.us-east-1.s3`
- **VPC**: your VPC
- **Subnets**: pick one private subnet in each of two AZs
- **Enable DNS name**: ✅ check this (still useful for any in-VPC resources)
- **Security group**: `sg-vpce-s3`
- **Policy**: Full access (tighten later)

CLI:

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxx \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.s3 \
  --subnet-ids subnet-aaa subnet-bbb \
  --security-group-ids sg-xxxxxxxx \
  --private-dns-enabled
```

Save the endpoint ID returned (e.g., `vpce-0abc123def456`).

---

## Step 3: Record the Endpoint ENI Private IPs

You'll put these in `/etc/hosts` later. Get them with:

```bash
aws ec2 describe-vpc-endpoints \
  --vpc-endpoint-ids vpce-0abc123def456 \
  --query 'VpcEndpoints[0].NetworkInterfaceIds' \
  --output text
```

That returns ENI IDs like `eni-0aaa... eni-0bbb...`. Get their private IPs:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-0aaa eni-0bbb \
  --query 'NetworkInterfaces[].PrivateIpAddress' \
  --output text
```

Example output: `10.0.1.23  10.0.2.45`

**Write these down.** Call them `ENI_IP_1` and `ENI_IP_2`.

---

## Step 4: Create the IAM User and Access Keys

AWS Console → IAM → Users → Create user:

- **Username**: `onprem-s3-uploader`
- Do **not** grant console access

Attach an inline policy (least privilege for a single bucket):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:AbortMultipartUpload",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-bucket"
    }
  ]
}
```

After the user is created → Security credentials → Create access key → Use case: "Application running outside AWS." Copy the **Access Key ID** and **Secret Access Key**. The secret is shown once — save it now.

---

## Step 5: Verify OCI VCN Routing and Security Lists Permit the Flow

On the OCI side, before touching the VM, confirm:

**VCN Route Table** (for the subnet hosting the VM): a route for `10.0.0.0/16` (AWS VPC CIDR) pointing to your DRG / IPsec attachment.

**VCN Security List or NSG** attached to the VM: egress rule allowing `TCP 443` to `10.0.0.0/16`.

Without these, nothing else will work. Quick test from the VM after it's up:

```bash
nc -zv 10.0.1.23 443
# Expect: Connection to 10.0.1.23 443 port [tcp/https] succeeded!
```

If that fails, fix routing/security lists before continuing.

---

## Step 6: Update `/etc/hosts` on the OCI VM

SSH into the Oracle Linux VM:

```bash
ssh opc@<your-vm-public-or-bastion-ip>
```

Back up the current hosts file:

```bash
sudo cp /etc/hosts /etc/hosts.bak
```

Edit:

```bash
sudo vi /etc/hosts
```

Add these lines (substitute `ENI_IP_1` with the IP you captured in Step 3, and `my-bucket` with your bucket name):

```
# AWS S3 Interface Endpoint (via VPN)
10.0.1.23   my-bucket.s3.us-east-1.amazonaws.com
10.0.1.23   s3.us-east-1.amazonaws.com
```

Save and close.

**Why only one IP per hostname?** `/etc/hosts` lookups don't load-balance across multiple entries reliably — the first match wins. Pick one ENI IP and accept that you don't get automatic failover to the other AZ. This is the trade-off of Option A, which is why it's a testing tool. For production, use dnsmasq for selective forwarding.

Verify:

```bash
getent hosts my-bucket.s3.us-east-1.amazonaws.com
# Should return: 10.0.1.23  my-bucket.s3.us-east-1.amazonaws.com
```

---

## Step 7: Verify Network Path to the Endpoint

```bash
# TCP reachability
nc -zv my-bucket.s3.us-east-1.amazonaws.com 443

# TLS handshake + HTTP response
curl -v https://my-bucket.s3.us-east-1.amazonaws.com
```

Expected: TCP connects, TLS handshake completes, S3 responds (likely a 403 XML body — that's fine, it means S3 is talking back).

If TCP fails: check OCI routing/security lists, AWS endpoint security group, and VPN tunnel status.

If TLS fails with a certificate error: this would be unusual — S3's certificate covers `*.s3.us-east-1.amazonaws.com`, which matches what we put in `/etc/hosts`. If you hit this, double-check you didn't typo the hostname.

---

## Step 8: Install AWS CLI v2 on the Oracle Linux VM

```bash
sudo dnf install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Expect something like `aws-cli/2.x.x Python/3.x.x Linux/...`.

---

## Step 9: Configure AWS Credentials

```bash
aws configure --profile onprem-uploader
```

Enter when prompted:

```
AWS Access Key ID [None]: AKIA....
AWS Secret Access Key [None]: ....
Default region name [None]: us-east-1
Default output format [None]: json
```

Lock down the credentials file:

```bash
chmod 600 ~/.aws/credentials
```

Quick identity check:

```bash
aws sts get-caller-identity --profile onprem-uploader
```

Returns your IAM user ARN. **Note**: this call goes to the STS public endpoint, not S3 — so it's traveling over OCI's internet egress, not the VPN. That's fine for now, but worth knowing. If you want STS to also flow over the VPN later, you'd add an STS interface endpoint and another `/etc/hosts` entry.

---

## Step 10: Upload a Test Object

```bash
echo "hello from OCI via VPN and S3 interface endpoint" > test.txt

aws s3 cp test.txt s3://my-bucket/test.txt --profile onprem-uploader
```

Expected:

```
upload: ./test.txt to s3://my-bucket/test.txt
```

List to confirm:

```bash
aws s3 ls s3://my-bucket/ --profile onprem-uploader
```

---

## Step 11: Prove the Upload Actually Used the Endpoint

Run with debug and look at the destination IP:

```bash
aws s3 cp test.txt s3://my-bucket/test2.txt \
  --profile onprem-uploader --debug 2>&1 \
  | grep -iE "establishing|connectionpool|resolved"
```

You should see the connection going to `10.0.1.23` (your endpoint ENI IP), not a public AWS IP range.

For stronger proof, tighten the bucket policy to require the endpoint:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnlessViaOurVPCEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-0abc123def456"
        }
      }
    }
  ]
}
```

Apply it in the S3 console under Bucket → Permissions → Bucket policy. After applying, re-run the upload — it should still succeed from the OCI VM. Any attempt to upload from outside the endpoint (e.g., from your laptop over the internet with the same credentials) will now be denied.

---

## Troubleshooting Quick Reference

| Symptom | Check |
|---|---|
| `Could not connect to the endpoint URL` | `getent hosts` returns right IP? `nc -zv <ip> 443` works? VPN tunnel up? |
| Connection times out | OCI security list egress 443 to AWS CIDR? AWS endpoint SG allows OCI CIDR? |
| TLS cert error | Typo in `/etc/hosts` hostname — must exactly match `<bucket>.s3.<region>.amazonaws.com` |
| `AccessDenied` | IAM policy wrong, or `aws:SourceVpce` in bucket policy doesn't match your endpoint ID |
| `NoSuchBucket` | Bucket name typo, or bucket is in a different region than your endpoint |
| Works from OCI, not from laptop | Expected — that's the bucket policy doing its job |

---

## What You'll Swap Later for IAM Roles Anywhere

Only Steps 4 and 9 change:
- Step 4 becomes: create a trust anchor, profile, and IAM role; issue a certificate to the VM
- Step 9 becomes: install the `aws_signing_helper`, configure a `credential_process` in `~/.aws/config`

Everything else — endpoint, `/etc/hosts`, networking — stays identical.