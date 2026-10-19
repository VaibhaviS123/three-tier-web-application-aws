# Three-Tier Web Application Architecture with AWS

## Project Overview

This project demonstrates the deployment of a three-tier web application architecture on Amazon Web Services (AWS). The architecture separates the **web tier**, **application tier**, and **database tier** to organize responsibilities and control how traffic flows between components.

I manually created and configured the EC2 instances for the web and application tiers. **This project does not use Auto Scaling Groups.** An internet-facing Application Load Balancer routes incoming web traffic to the web-tier instances, while an internal Application Load Balancer forwards application requests to the application-tier instances.

AWS Systems Manager (SSM) Session Manager is used to access EC2 instances without requiring a bastion host or inbound SSH access. Application code and configuration files are stored in a private S3 bucket for deployment to the instances.

## Architecture Overview

![Three-Tier AWS Architecture](screenshots/architecture.png)

> Add your architecture diagram to the `screenshots` folder and name it `architecture.png`, or update the image path above to match your screenshot filename.

### Request Flow

1. A user sends a request to the internet-facing Application Load Balancer.
2. The external load balancer forwards the request to a healthy registered web-tier EC2 instance.
3. Nginx on the web tier serves web content and forwards applicable API requests to the internal Application Load Balancer.
4. The internal load balancer forwards those requests to a registered application-tier EC2 instance.
5. The application tier communicates with Amazon RDS for MySQL in the private database subnet.
6. The response travels back through the application tier and web tier to the user.

## AWS Services Used

| Service | Purpose |
|---|---|
| Amazon VPC | Provides an isolated network for project resources. |
| Public and private subnets | Separate internet-facing components from application and database resources. |
| Availability Zones | Distribute resources across the configured zones, where applicable. |
| Internet Gateway | Provides internet connectivity for resources in appropriately configured public subnets. |
| NAT Gateway | Provides outbound internet access to private-subnet resources, if configured in the deployment. |
| Amazon EC2 | Hosts the Nginx web servers and application servers. Instances were created manually. |
| Elastic Load Balancing — Application Load Balancer | Routes incoming traffic to the web tier and application requests to the app tier. |
| Amazon RDS for MySQL | Hosts the relational database in the database tier. |
| Amazon S3 | Stores application code and configuration files used during deployment. |
| AWS Systems Manager Session Manager | Provides managed access to EC2 instances without a bastion host or inbound SSH rule. |
| AWS IAM | Controls permissions for AWS resources and EC2 instance access. |
| Security Groups | Restrict traffic between the load balancers, EC2 instances, and database. |

> Keep only the services and configurations that you actually deployed. Remove any optional service or networking component that was not part of your implementation.

## Network and Security Design

- Created a VPC to isolate the project network.
- Used public and private subnets to separate the web, application, and database layers according to the deployment design.
- Configured an internet-facing Application Load Balancer for incoming web traffic.
- Configured an internal Application Load Balancer for traffic between the web tier and application tier.
- Used security groups to limit access to the required ports and sources.
- Placed the RDS for MySQL database in private networking and allowed database traffic only from the appropriate application-tier security group.
- Used AWS Systems Manager Session Manager to access EC2 instances instead of deploying a bastion host.
- Used IAM permissions to allow the required instance-level access to AWS services, including S3 and Systems Manager where configured.

## EC2 Instance Deployment

### Web Tier

- Created EC2 instances for the web tier manually; no Auto Scaling Group was configured.
- Installed and configured Nginx as the web server.
- Configured Nginx to forward applicable API requests to the internal Application Load Balancer.
- Registered the web instances with the target group associated with the internet-facing load balancer.

### Application Tier

- Created EC2 instances for the application tier manually; no Auto Scaling Group was configured.
- Configured the application and its database connection settings.
- Registered the application instances with the target group associated with the internal Application Load Balancer.
- Configured the application tier to communicate with the RDS for MySQL database through the required network and security-group rules.

### Database Tier

- Deployed Amazon RDS for MySQL in the database tier.
- Configured database access so that application instances could connect using the database endpoint, credentials, and required port.
- Kept database access restricted through network placement and security-group rules.

## Application Deployment with Amazon S3

1. Created a private S3 bucket to store the application code and configuration files.
2. Updated the relevant configuration files before uploading them, including the application database configuration and Nginx configuration where required.
3. Accessed the relevant EC2 instances through SSM Session Manager.
4. Downloaded the required files from S3 to the web or application instances.
5. Installed or configured the required software and started the relevant services.
6. Tested the request flow through the load balancers and verified connectivity between the application tier and database.

Do not upload database passwords, access keys, private keys, or other secrets to a public repository. For a production design, use a secrets-management solution such as AWS Secrets Manager or Systems Manager Parameter Store.

## Secure EC2 Access with SSM Session Manager

Instead of using a bastion host, I used AWS Systems Manager Session Manager to access the EC2 instances.

This approach avoids the need to expose inbound SSH access solely for administration. The instances must have the required SSM agent, IAM instance permissions, and network connectivity to Systems Manager endpoints.

## Load Balancer Health Checks

The Application Load Balancers use target groups to route traffic to registered instances. Health checks help determine whether a target is healthy according to the configured protocol, port, and path. The load balancer can stop routing new requests to targets that fail health checks.

Health checks do not create replacement EC2 instances by themselves. Since this project does not use Auto Scaling Groups, failed instances must be investigated and recovered or replaced manually.

## Screenshots

Add your own screenshots to the `screenshots/` directory and update the filenames below to match your files.

| Screenshot | Suggested filename |
|---|---|
| Architecture diagram | `screenshots/architecture.png` |
| VPC and subnet configuration | `screenshots/vpc-subnets.png` |
| EC2 web and app instances | `screenshots/ec2-instances.png` |
| Internet-facing load balancer | `screenshots/external-alb.png` |
| Internal load balancer | `screenshots/internal-alb.png` |
| Target group health status | `screenshots/target-groups.png` |
| RDS database configuration | `screenshots/rds-mysql.png` |
| S3 bucket with deployment files | `screenshots/s3-deployment-files.png` |
| SSM Session Manager connection | `screenshots/ssm-session-manager.png` |
| Working application in browser | `screenshots/application-working.png` |

Example Markdown for displaying a screenshot:

```markdown
![EC2 Instances](screenshots/ec2-instances.png)
```

## Challenges and Learnings

- Understanding traffic flow between the internet-facing load balancer, web tier, internal load balancer, application tier, and database.
- Configuring security groups so each tier can communicate only with the required components.
- Connecting the application tier to Amazon RDS for MySQL.
- Using S3 to store and retrieve deployment files.
- Using SSM Session Manager for instance administration without a bastion host.
- Understanding how load balancer health checks affect traffic routing and why health checks alone do not replace failed EC2 instances.

## Current Project Scope

- **Implemented:** manually created EC2 instances for the web and application tiers.
- **Implemented:** internet-facing and internal Application Load Balancers, as configured in the deployment.
- **Implemented:** SSM Session Manager in place of a bastion host.
- **Implemented:** S3-based storage and retrieval of application/configuration files.
- **Not implemented:** Auto Scaling Groups.

## Possible Future Improvements

- Configure Auto Scaling Groups for automatic instance replacement and scaling.
- Add HTTPS using an appropriate TLS certificate and listener configuration.
- Store credentials in AWS Secrets Manager or Systems Manager Parameter Store.
- Add centralized logging, alarms, and monitoring if required.
- Automate deployment with a CI/CD pipeline.
- Review cost and remove unused AWS resources after testing.

## Cleanup

To avoid unexpected AWS charges after testing, review and delete resources that are no longer needed. Depending on what you created, this may include:

1. EC2 instances.
2. Application Load Balancers and target groups.
3. Amazon RDS database resources.
4. NAT Gateways and allocated Elastic IP addresses, if used.
5. S3 objects and buckets that are no longer required.
6. Unused VPC resources, route tables, and security groups.

Before deleting resources, back up any data or screenshots you want to keep and verify that the resources are no longer needed.

## Disclaimer

This is a learning project built to practice AWS networking, compute, load balancing, database connectivity, deployment, and secure instance access. It is not a claim of a production-ready deployment. The actual availability and security characteristics depend on the resources and settings configured in the AWS account.
