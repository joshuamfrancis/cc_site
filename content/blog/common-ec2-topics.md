---
title: "Common Ec2 Topics"
date: 2026-04-03T19:58:39+10:00
draft: true
tags: ["hugo", "static-sites", "tutorial"]
categories: ["EC2"]
featured_image: ""
summary: "Common EC2 vm provisioning topics"
---

# Common EC2 provisioning topics

## User Data for installing SSM-Agent in Ubuntu
```bash
#!/bin/bash
# Install and start SSM Agent on Ubuntu EC2 via apt
mkdir -p /tmp/ssm
cd /tmp/ssm
REGION=$(ec2metadata --availability-zone | sed 's/[a-z]$//')
wget "https://s3.${REGION}.amazonaws.com/amazon-ssm-${REGION}/latest/debian_amd64/amazon-ssm-agent.deb"
dpkg -i amazon-ssm-agent.deb
systemctl enable amazon-ssm-agent
systemctl start amazon-ssm-agent
```