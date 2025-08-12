If you’re installing Neo4j in AWS with a private IP setup (no public internet access), you’ll need to make sure the core services, dependencies, and supporting tools are available internally so the database can run and be monitored, backed up, and administered without internet.

Here’s the recommended service checklist for such a setup:


---

1. Core Neo4j Services

These are part of the database installation itself:

Neo4j Core Servers – for the database cluster nodes (primary write nodes).

Neo4j Read Replica Servers – optional, for horizontal read scaling.

Neo4j Browser / Bloom – browser-based UI and visualization tools.

Neo4j Licensing Service – if using enterprise edition, ensure license file is stored locally.

Bolt Protocol Endpoint – for app-to-database connections.



---

2. Java Runtime

Neo4j requires Java (Java 17 or higher for latest versions):

Amazon Corretto or OpenJDK installed locally.

Since internet is blocked, have the .rpm or .tar.gz package available in S3 or JFrog Artifactory.



---

3. Internal Package Repository

For offline installation:

YUM/DNF local repo (Amazon Linux/RHEL/CentOS) or APT local repo (Debian/Ubuntu) in your VPC.

Store Neo4j .deb / .rpm or .tar.gz files in S3, JFrog, or EFS.



---

4. Cluster Coordination & Networking

DNS service – AWS Route 53 private hosted zone or Consul for internal name resolution between nodes.

NTP service – AWS Time Sync Service or internal NTP server (cluster requires synced clocks).

VPC & Subnet Configuration – private subnets in multiple AZs for HA.

Security Groups – open required Neo4j ports only inside VPC:

7687 → Bolt

7474 → HTTP API

7473 → HTTPS

Internal gossip ports if in a cluster.




---

5. Monitoring Services

Prometheus Node Exporter – for OS-level metrics.

Neo4j Prometheus Plugin – for DB metrics (comes with Neo4j Enterprise).

Grafana – dashboard for metrics visualization.

CloudWatch Agent (optional) – send logs & metrics to AWS CloudWatch.



---

6. Backup & Recovery Services

Neo4j Backup Tool (neo4j-admin backup) installed on a dedicated node or jump host.

S3 VPC Endpoint – to store backups directly into an S3 bucket without public internet.

EFS or FSx (optional) – for shared backup storage in private network.



---

7. Authentication & Access Control

LDAP/AD integration service – for enterprise authentication (requires private connectivity to AD/LDAP).

IAM roles & policies – for S3 backup/restore, EC2 role for instance permissions.

SSL/TLS Certificates – stored in AWS Secrets Manager or internal CA.



---

8. Automation & Deployment Services

Terraform / Ansible – stored in a private Git repo or CodeCommit.

S3 VPC Endpoint – to fetch software artifacts and configs.

JFrog Artifactory – if storing .tar.gz or .rpm for Neo4j & Java.



---

💡 Flow in a private IP setup:

1. EC2 nodes in private subnets get software from S3 (via VPC endpoint) or JFrog internal URL.


2. All cluster communication happens over private VPC DNS and internal subnets.


3. Monitoring, backup, and IAM are done via internal AWS services only — no public internet needed.




---

If you want, I can prepare you a full AWS Neo4j private VPC architecture diagram + service list so you know exactly which AWS components to enable for this. That would make it crystal clear.

---Neo4j on AWS (Private IP) — Terraform Project

Purpose: This page documents the Terraform project used to deploy a highly-available Neo4j cluster in AWS using private IPs only (no public internet access). It is intended as an internal wiki reference for engineers to understand, deploy, operate, and troubleshoot the environment.


---

Table of Contents

1. Project Summary


2. Architecture Diagram (high-level)


3. Repo Layout & File Details


4. Terraform module descriptions


5. Environment-specific configuration


6. Example Terraform files (snippets)


7. Bootstrap & userdata scripts (overview + examples)


8. CI/CD: GitLab pipeline example


9. Backup & Restore runbook


10. Monitoring & Alerting


11. Security checklist


12. Operational runbooks (start/stop, scale, upgrade)


13. Troubleshooting guide


14. Change log / Notes




---

1. Project Summary

Goal: Deploy Neo4j Enterprise cluster across multiple AZs in a private VPC with S3-based backups, Prometheus/Grafana monitoring, and tight IAM/security controls.

Assumptions:

No public IPv4 addresses on Neo4j nodes.

Artifact store (JFrog or S3) is reachable via VPC endpoints.

Terraform remote state lives in S3 with DynamoDB locking.




---

2. Architecture Diagram (high-level)

> Add an architecture diagram image here in the wiki. Suggested components:



VPC (private subnets in 3 AZs)

EC2 Auto-scaling groups or fixed EC2 for Neo4j cores and read replicas

Route53 private hosted zone for cluster DNS

S3 bucket for backups (with VPC endpoint)

Prometheus + Grafana in private subnet

IAM roles & policies for EC2 instances

SSM Session Manager for access



---

3. Repo Layout & File Details

terraform-neo4j-private-vpc/
├── README.md
├── versions.tf
├── providers.tf
├── backend.tf
├── variables.tf
├── outputs.tf
├── main.tf
├── modules/
│   ├── vpc/
│   ├── security/
│   ├── ec2-neo4j/
│   ├── s3-backup/
│   ├── monitoring/
│   └── dns/
├── scripts/
│   ├── install_neo4j.sh
│   ├── configure_cluster.sh
│   ├── install_prometheus.sh
│   └── backup_restore.sh
├── envs/
│   ├── dev/terraform.tfvars
│   ├── staging/terraform.tfvars
│   └── prod/terraform.tfvars
└── .gitlab-ci.yml

Short descriptions:

versions.tf — Terraform & provider version pinning.

providers.tf — AWS provider config and region/provider-level settings.

backend.tf — S3 backend + DynamoDB lock config.

variables.tf / outputs.tf — global vars and outputs used by root module.

modules/ — logically separated components (VPC, security, EC2, S3 backups, monitoring, DNS).

scripts/ — scripts used by user_data or provisioners to bootstrap nodes offline from S3/JFrog.

envs/ — tfvars and backend overrides per environment.

.gitlab-ci.yml — pipeline to plan and apply with appropriate approvals.



---

4. Terraform Module Descriptions

modules/vpc

Creates VPC, private subnets across multiple AZs

Creates route tables, NATs (optional), VPC endpoints (S3, SSMMessaging)

Outputs: vpc_id, private_subnet_ids, azs


modules/security

Security groups for Neo4j (bolt, http/https), monitoring, backups

Network ACL templates (optional)

IAM roles and policies for EC2 (S3 access, CloudWatch, SSM)

Outputs: neo4j_sg_id, instance_profile_arn


modules/ec2-neo4j

Launch templates or instance resources for Neo4j nodes

Attach IAM instance profile

Provide user_data that pulls installers from a private artifact store

Optionally uses cloud-init to run scripts/install_neo4j.sh

Outputs: instance_ids, private_ips, private_dns_names


modules/s3-backup

Creates S3 bucket with encryption, versioning, lifecycle rules

Creates S3 bucket policy restricted to VPC endpoints and roles

Outputs: backup_bucket_name


modules/monitoring

Node exporter installation (via userdata/script) on Neo4j nodes

Standalone Prometheus + Grafana EC2 or ECS/Fargate in private subnets

Outputs: prometheus_endpoint, grafana_endpoint


modules/dns

Creates Route 53 private hosted zone

Adds A/CNAME records for each Neo4j node (private IPs)



---

5. Environment-specific configuration

Each envs/<env>/terraform.tfvars contains values like:

region = "ap-south-1"
cluster_name = "neo4j-prod"
node_count_core = 3
instance_type_core = "m5.large"
instance_type_read = "m5.large"
vpc_cidr = "10.10.0.0/16"
private_subnet_cidrs = ["10.10.1.0/24", "10.10.2.0/24", "10.10.3.0/24"]
backup_bucket_name = "neo4j-prod-backups"
artifact_bucket = "internal-artifacts"
artifactory_url = "https://jfrog.internal.local/artifactory/neo4j"

Keep secrets and credentials in the CI/CD secret store (GitLab CI variables, or Vault/Secrets Manager). Do not commit credentials to repo.


---

6. Example Terraform files (snippets)

versions.tf

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws" version = ">= 5.0" }
  }
}

providers.tf

provider "aws" {
  region = var.region
  # Recommended: use environment credentials or assume-role behavior in CI
}

backend.tf (example using S3 + DynamoDB)

terraform {
  backend "s3" {
    bucket = "terraform-state-bucket"
    key    = "neo4j/${var.env}/terraform.tfstate"
    region = var.region
    dynamodb_table = "terraform-locks"
    encrypt = true
  }
}

main.tf (root module call example)

module "vpc" {
  source = "./modules/vpc"
  vpc_cidr = var.vpc_cidr
  azs = var.azs
  private_cidrs = var.private_subnet_cidrs
}

module "security" {
  source = "./modules/security"
  vpc_id = module.vpc.vpc_id
}

module "ec2_neo4j" {
  source = "./modules/ec2-neo4j"
  vpc_id = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  instance_profile = module.security.instance_profile_arn
  node_count = var.node_count_core
}


---

7. Bootstrap & userdata scripts (overview + examples)

Store all scripts under scripts/ and serve them from an internal Artifactory or S3 bucket accessible via VPC endpoint.

scripts/install_neo4j.sh (simplified example)

#!/bin/bash -eux
ARTIFACT_URL="$1"    # passed from user_data or userdata templating
# install java
mkdir -p /opt/neo4j-install
aws s3 cp "${ARTIFACT_URL}/corretto-17.tar.gz" /opt/neo4j-install/
# extract & install corretto (example)
cd /opt/neo4j-install
tar xzf corretto-17.tar.gz -C /usr/lib/jvm
# install neo4j
aws s3 cp "${ARTIFACT_URL}/neo4j-enterprise.tar.gz" /opt/neo4j-install/
tar xzf neo4j-enterprise.tar.gz -C /opt/
# setup neo4j.conf, set dbms.connectors.default_listen_address=0.0.0.0 etc.
# configure TLS certificates (pull from Secrets Manager if available)
# start service
/opt/neo4j/bin/neo4j start

> Note: In production make sure scripts are idempotent, handle retries, and log to /var/log/.



neo4j.conf — key config entries (essential)

# Network
dbms.default_listen_address=0.0.0.0
dbms.connector.bolt.listen_address=:7687
dbms.connector.http.listen_address=:7474
dbms.connector.https.listen_address=:7473

# HA & clustering
causal_clustering.discovery_listen_address=0.0.0.0:5000
causal_clustering.transaction_listen_address=0.0.0.0:6000
causal_clustering.cluster_listen_address=0.0.0.0:7000

# Auth
dbms.security.auth_enabled=true

# Monitoring (Prometheus exporter)
# metrics.prometheus.enabled=true
# metrics.prometheus.endpoint=/metrics


---

8. CI/CD: GitLab pipeline example (.gitlab-ci.yml)

stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: "./"

validate:
  stage: validate
  script:
    - terraform init -backend-config=envs/$ENV/backend.tfvars
    - terraform validate
  only:
    - merge_requests

plan:
  stage: plan
  script:
    - terraform init -backend-config=envs/$ENV/backend.tfvars
    - terraform plan -var-file=envs/$ENV/terraform.tfvars -out=tfplan
  artifacts:
    paths:
      - tfplan
  only:
    - branches

apply:
  stage: apply
  when: manual
  script:
    - terraform init -backend-config=envs/$ENV/backend.tfvars
    - terraform apply -auto-approve tfplan
  only:
    - master

Secrets & credentials:

Use GitLab CI/CD variables for AWS credentials / role assumption ARN.

Use Vault/Secrets Manager integration for sensitive values.



---

9. Backup & Restore runbook

Backup (automated)

Use neo4j-admin backup (enterprise) to create physical backups.

Backups should be pushed to S3 bucket via the instance IAM role.

Schedule via cron system user on dedicated backup node or via SSM Automation document.


Example backup command:

/opt/neo4j/bin/neo4j-admin backup --backup-dir=/backups/neo4j --pagecache=4G --to=offline
# then aws s3 cp /backups/neo4j s3://${BACKUP_BUCKET}/${CLUSTER_NAME}/$(date +%F)/ --recursive

Restore (high-level)

1. Provision a replacement node in the same AZ/subnet using ec2-neo4j module.


2. Stop neo4j on that node.


3. Download the backup from S3 and run neo4j-admin restore per Neo4j docs.


4. Start the node, ensure it joins cluster and syncs.



Important: Test restore procedures quarterly and keep a documented checklist for RTO/RPO expectations.


---

10. Monitoring & Alerting

Install Prometheus Node Exporter on every Neo4j instance (via userdata).

Use Neo4j Prometheus exporter or built-in exporter for DB metrics.

Prometheus scrape targets should be private endpoints in the VPC.

Grafana for dashboards (save dashboards as JSON in repo for reproducibility).

Alerts: Disk usage, JVM memory, page cache usage, unavailable cores, high replication lag.


Sample alerts:

instance_cpu_utilization > 85% for 5m

neo4j_transaction_commit_latency > 500ms



---

11. Security checklist

VPC private subnets for DB nodes; no public IPs.

Minimum required ports in security groups: 7687 (Bolt), 7473/7474 (HTTP/HTTPS), cluster gossip ports.

S3 bucket policies restrict access to the cluster role and VPC endpoint only.

Use IAM roles for EC2 (no long-lived keys on instances).

Store TLS certs in AWS Secrets Manager or internal Vault; rotate periodically.

Enable AWS Config / CloudTrail for audit.

Enable encryption-at-rest for volumes (EBS) and backups (S3 SSE).



---

12. Operational runbooks

Start/Stop Neo4j node

# start
sudo /opt/neo4j/bin/neo4j start
# stop
sudo /opt/neo4j/bin/neo4j stop

Scale out (add a core)

1. Increase node_count_core in envs/<env>/terraform.tfvars.


2. Run terraform apply to create new instance(s).


3. Ensure new node bootstraps and joins the cluster via logs /var/log/neo4j/.



Rolling upgrade

1. Upgrade cluster one node at a time.


2. Drain reads from node (if applicable), stop neo4j on node, update packages, start, verify.


3. Repeat for each node.




---

13. Troubleshooting guide

Node not joining cluster: check neo4j.log and debug.log for discovery/gossip errors; confirm Route53 private DNS resolves other nodes.

Metrics missing in Prometheus: verify node exporter is running, check scrape configs, ensure security group allows Prometheus source.

Backups failing to S3: check IAM role permissions and VPC endpoint policies; test aws s3 cp from instance.

High heap usage / OOM: inspect JVM flags, increase heap or tune pagecache.



---

14. Change log / Notes

Maintain a CHANGELOG.md in the repo. Record infra changes, Terraform version bumps, module upgrades, and tested dates for restores.



---

Appendix: Useful Commands

terraform init -backend-config=envs/prod/backend.tfvars

terraform plan -var-file=envs/prod/terraform.tfvars

terraform apply -var-file=envs/prod/terraform.tfvars

aws s3 ls s3://neo4j-prod-backups/ (run from within VPC using proper role)



---

End of document.

If you want this exported to Confluence markup, or want me to add the architecture diagram file (SVG/PNG) and copy Terraform module templates into the repo, say “Export to Confluence” or “Generate module templates” and I’ll prepare them.

