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

