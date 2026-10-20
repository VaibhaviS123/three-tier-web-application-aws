# AWS Three Tier Web Application Architecture

## Description:

This project is a hands-on implementation of a three-tier web application architecture on AWS. It demonstrates how to configure networking, security groups, manually launched EC2 instances, Application Load Balancers, Amazon RDS for MySQL, Amazon S3, and AWS Systems Manager Session Manager to deploy and manage a web application.

## Audience:

It is intended for those who have a technical role. The assumption is that you have at least some foundational aws knowledge around VPC, EC2, RDS, S3, ELB and the AWS Console.

## Pre-requisites:

1. An AWS account. If you don’t have an AWS account, follow the instructions [here](https://aws.amazon.com/console/) and
   click on “Create an AWS Account” button in the top right corner to create one.
1. IDE or text editor of your choice.

## Architecture Overview

This project implements a three-tier web application architecture on AWS using manually launched EC2 instances, Application Load Balancers, Amazon RDS for MySQL, Amazon S3, and AWS Systems Manager Session Manager.

The external Application Load Balancer routes incoming requests to the web tier, where Nginx serves the frontend and forwards API requests to the internal Application Load Balancer. The internal load balancer distributes requests across the registered application-tier EC2 instances. The application servers communicate with the private RDS MySQL database to retrieve and manage application data.

Security groups control traffic between the tiers, while the VPC, subnets, and routing configuration provide network separation. AWS Systems Manager Session Manager provides administrative access to EC2 instances without requiring a bastion host.

The EC2 instances were launched and configured manually. An Auto Scaling Group was not implemented in this project.
<img width="2048" height="1383" alt="Vaibhavi Shinde" src="https://github.com/user-attachments/assets/00963073-c131-4b30-8193-15796f28a154" />



## Request Flow

1. **User Request:** A user accesses the application through the external Application Load Balancer (EALB).
2. **External Load Balancer:** The EALB forwards incoming requests to the web-tier EC2 instance or instances registered with it.
3. **Web Tier:** Nginx serves the frontend content and forwards application requests to the internal Application Load Balancer (IALB).
4. **Internal Load Balancer:** The IALB distributes incoming application requests among the registered application-tier EC2 instances.
5. **Application Tier:** The application servers process requests and communicate with the database when data access is required.
6. **Database Tier:** Amazon RDS for MySQL stores and retrieves application data. The database should accept MySQL connections only from the authorized application-tier security group.
7. **Administration:** AWS Systems Manager Session Manager provides access to EC2 instances without requiring a bastion or jump server.
8. **Deployment Files:** Amazon S3 stores the application deployment files used to configure the EC2 instances.
## Setup - Part 0

### Download Code from Github

git clone https://github.com/vaibhavis123/aws-three-tier-web-app.git

### S3 Bucket Creation

1. Navigate to the S3 service in the AWS console and create a new S3 bucket.

2. Select the region you want and give unique name to the bucket

### IAM EC2 Instance Role Creation

1. Navigate to the IAM dashboard in the AWS console and create an EC2 role.
2. Select EC2 as the trusted entity.

3. When adding permissions, include the following AWS managed policies. You can search for them and select them. These policies will allow our instances to download our code from S3 and use Systems Manager Session Manager to securely connect to our instances without SSH keys through the AWS console.

   - AmazonSSMManagedInstanceCore
   - AmazonS3ReadOnlyAccess

4. Give your role a name, and then click Create Role.

## Networking and Security - Part 1

Now we will be building out the VPC networking components as well as security groups that will add a layer of protection around our EC2 instances, RDS MySQL databases, and Elastic Load Balancers.

- Create an isolated network with the following components:
  - VPC
  - Subnets
  - Route Tables
  - Internet Gateway
  - NAT gateway
  - Security Groups

### VPC and Network Configuration

#### VPC Creation

1. Navigate to the Amazon VPC dashboard in the AWS Management Console.
2. Click **Create VPC** and select the **VPC and more** option.
3. Provide a suitable name for the VPC and configure the IPv4 CIDR block according to the network requirements of the three-tier architecture.
4. Configure the required number of Availability Zones and subnets for the web, application, and database layers.
5. Review the network configuration and create the VPC.

**Note:** The VPC and more option automatically creates and configures several networking components based on the selected settings, reducing the need to create each component manually.

<img width="933" height="456" alt="Screenshot 2026-10-18 235322" src="https://github.com/user-attachments/assets/af0e5e05-bc43-4196-a1f1-04a9a325a417" />
<img width="935" height="451" alt="Screenshot 2026-10-18 235352" src="https://github.com/user-attachments/assets/27c26a67-2dc9-434b-8adc-eef3311914bb" />
<img width="937" height="450" alt="Screenshot 2026-10-18 235744" src="https://github.com/user-attachments/assets/571141f8-2732-46ff-8eed-aca3ea8fa073" />
![Uploading Screenshot 2026-10-18 235744.png…]()

#### Subnet Configuration

The VPC was configured to support the three-tier application architecture, separating the web, application, and database layers into appropriate subnets.

The subnet configuration provides network isolation between the different application tiers and allows traffic to be controlled through route tables and security groups.

- **Public subnets:** Used for resources that require public connectivity, including the external Application Load Balancer and web-tier entry point.
- **Private application subnets:** Used to host the application-tier EC2 instances, which process requests forwarded by the internal load balancer.
- **Private database subnets:** Used to provide isolated network placement for the Amazon RDS MySQL database.

#### Internet Connectivity

**Internet Gateway (IGW)**

An Internet Gateway is attached to the VPC to provide internet connectivity to resources in public subnets, subject to their routing and security configurations.

**NAT Gateway**

The NAT Gateway provides outbound internet connectivity to resources in private subnets when required, without allowing unsolicited inbound internet connections to those resources.

The required NAT Gateway configuration was selected during VPC creation using the VPC and more workflow.

#### Routing Configuration

Route tables control how network traffic moves between subnets and external networks.

- **Public route table:** Routes internet-bound traffic through the Internet Gateway.
- **Private application route tables:** Route eligible outbound internet traffic through the NAT Gateway.
- **Private database route table:** Keeps the database tier isolated from direct internet routing while allowing the required communication within the VPC.

The VPC and more workflow created the initial networking configuration based on the selected options. The resulting route tables and subnet associations can be reviewed in the VPC console.

#### Network Architecture

The VPC provides the network foundation for the three-tier application. Public and private subnet placement, routing, and security groups work together to control access between the external load balancer, web tier, internal load balancer, application tier, and database tier.

### Security Groups

Five security groups are configured to control communication between the external load balancer, web tier, internal load balancer, application tier, and database tier.

#### 1. External Application Load Balancer — EALB Security Group

Configure the external Application Load Balancer using the **EALB Security Group**. Allow inbound HTTP (port 80) and/or HTTPS (port 443) traffic from the intended client IP ranges. For a publicly accessible application, configure appropriate public inbound access; for a restricted demonstration, limit access to your IP address.

The external load balancer forwards incoming requests to the web-tier target group.

<img width="1892" height="913" alt="External ALB Security Group" src="https://github.com/user-attachments/assets/83ccf72f-2541-4eca-8910-ada4e89f979a" />

<img width="936" height="451" alt="EALB Inbound Rules" src="https://github.com/user-attachments/assets/63b1d973-a9ac-4e93-aabe-603d84dbd619" />

#### 2. Web Server Security Group

Configure the **Web Server Security Group** for the EC2 instance running Nginx. Allow inbound HTTP traffic from the **EALB Security Group** so that requests forwarded by the external load balancer can reach the web server.

If you added a separate rule allowing direct HTTP access from your IP for testing, document it only if that rule is still configured. For a more restricted setup, remove the direct-access rule when it is no longer needed.

<img width="932" height="451" alt="Web Server Security Group" src="https://github.com/user-attachments/assets/b47dc9ad-c0c2-4258-910b-e6cc40e90676" />

#### 3. Internal Application Load Balancer — IALB Security Group

Configure the **IALB Security Group** for the internal Application Load Balancer. Allow inbound application traffic from the **Web Server Security Group**. This permits the web tier to forward application requests to the internal load balancer.

<img width="940" height="492" alt="IALB Security Group" src="https://github.com/user-attachments/assets/46c89a28-a082-4c8b-91bc-98f087aef878" />

#### 4. Application Server Security Group

Configure the **App Server Security Group** for the application-tier EC2 instances. Allow inbound TCP traffic on port `4000` from the **IALB Security Group**, provided that port 4000 is the port on which your application listens.

If a direct-access rule was added for testing, document it only if it is actually configured. In the final setup, application traffic should normally reach the instances through the internal load balancer rather than directly from the internet.

<img width="935" height="452" alt="App Server Security Group" src="https://github.com/user-attachments/assets/aef3357c-345e-4ba7-b6d9-ef03afab1902" />

#### 5. Database Security Group

Configure the **Database Security Group** for Amazon RDS for MySQL. Allow inbound MySQL traffic on TCP port `3306` from the **App Server Security Group**. This permits the application servers to connect to the database while restricting database access to the authorized application tier.

<img width="927" height="452" alt="Database Security Group" src="https://github.com/user-attachments/assets/1114adf8-28f2-4b9d-a18e-77f5d903baf2" />




## Database Deployment - Part 2

- Deploy Database Layer
  - Subnet Groups
  - Multi-AZ Database

### Subnet Groups

1. Navigate to the RDS dashboard in the AWS console and click on Subnet groups on the left hand side. Click Create DB subnet group.
2. Give your subnet group a name, description, and choose the VPC we created.
   <img width="1917" height="901" alt="Screenshot 2026-10-19 001938" src="https://github.com/user-attachments/assets/50d38e03-8288-4806-a8c7-ebe201c2ce1b" />

3. When adding subnets, make sure to add the subnets we created in each availability zone specificaly for our database layer. You may have to navigate back to the VPC dashboard and check to make sure you're selecting the correct subnet IDs.
      ![Uploading Screenshot 2026-10-19 001938.png…]()

### Database Deployment – Amazon RDS for MySQL

#### RDS MySQL Database Creation

1. Navigate to the **Amazon RDS** dashboard in the AWS Management Console and select **Databases**.
2. Click **Create database** and select **Standard create**.
3. Under Engine options, select **MySQL** as the database engine.
4. Choose an appropriate database template based on your project requirements.
5. Configure the DB instance identifier, master username, and password.
6. Under **Connectivity**, select the VPC used for the three-tier application and configure the appropriate DB subnet group.
7. Disable public access to keep the database private.
8. Review the configuration and click **Create database**.

**RDS Database Configuration Screenshot**

<img width="1905" height="382" alt="Screenshot 2026-10-19 002053" src="https://github.com/user-attachments/assets/d4b2c047-a431-4ec6-a8e3-1d81d2e1dff4" />

<img width="1917" height="903" alt="Screenshot 2026-10-19 002118" src="https://github.com/user-attachments/assets/88d6166e-b4bd-4086-b432-db59ba66e6eb" />

<img width="1912" height="906" alt="Screenshot 2026-10-19 002221" src="https://github.com/user-attachments/assets/8349edb7-451e-4f01-9b81-09ea583aeb7a" />

#### Database Connectivity and Security

The Amazon RDS for MySQL database acts as the database layer of the three-tier architecture. It stores application data and communicates with the application-tier EC2 instances through the private network.

The following security configurations help protect the database:

- **Private Database Access:** Public accessibility is disabled to prevent direct access from the internet.
- **Security Groups:** Inbound MySQL traffic is allowed only from the application server security group.
- **Database Authentication:** The application uses the configured database credentials to establish a connection.
- **Network Isolation:** The database is deployed within the VPC using the configured DB subnet group.

**RDS Connectivity and Security Screenshot**

<img width="1900" height="908" alt="Screenshot 2026-10-19 002246" src="https://github.com/user-attachments/assets/e279cf05-e2be-4681-9679-f5318ce8e25a" />
<img width="1917" height="906" alt="Screenshot 2026-10-19 002313" src="https://github.com/user-attachments/assets/77837e0f-cfe6-4af9-93d3-12916210a5c6" />


#### Database Endpoint Configuration

After the database becomes available, navigate to the database details page in the Amazon RDS console and locate the endpoint and port.

The RDS endpoint is used by the application server to establish a connection to the MySQL database. The endpoint and database credentials are configured in the application's database configuration file, such as `database.conf`.

**RDS Endpoint Screenshot**


<img width="1916" height="901" alt="Screenshot 2026-10-19 002821" src="https://github.com/user-attachments/assets/83a068d5-e668-4998-b3df-0228c4aa6317" />



#### Database Communication Flow

1. The user sends a request through the external Application Load Balancer.
2. The request reaches the web tier, where Nginx forwards it to the internal Application Load Balancer.
3. The internal load balancer forwards the request to a healthy application server.
4. The application server connects to Amazon RDS for MySQL over the private network.
5. The database processes the query and returns the result to the application.
6. The application response travels back through the web tier and external load balancer to the user.


## App Tier Instance Deployment - Part 3


In this section, we will create an EC2 instance for our app layer and make all necessary software configurations so that the app can run. The app layer consists of a Node.js application that will run on port 4000. We will also configure our database with some data and tables.

- Learning Objectives:
  - Create App Tier Instance
  - Configure Software Stack
  - Configure Database Schema
  - Test DB connectivity



### Learning Objectives

- Launch and configure App Tier EC2 instances manually.
- Connect to instances securely using AWS Systems Manager Session Manager.
- Install the MySQL client and verify database connectivity.
- Configure the database schema and test database operations.
- Download application code from a private Amazon S3 bucket.
- Install the required runtime and dependencies.
- Start and verify the backend application.
- Configure the application instances to receive traffic through the internal load balancer.

---

### App Instance Deployment

1. Navigate to the **Amazon EC2** dashboard in the AWS Management Console and select **Instances**.
2. Click **Launch instances** and select the appropriate Amazon Machine Image (AMI), such as Amazon Linux 2.
3. Choose an appropriate EC2 instance type based on the application's requirements.
4. Configure the network settings by selecting the project VPC and an appropriate private application subnet.
5. Configure the instance details and attach the required IAM instance profile to support AWS Systems Manager and Amazon S3 access.
6. Add a suitable name tag, such as `AppServer`.
7. Select the App Server Security Group and review the configuration.
8. Launch the instance and verify that it reaches the running state.

Repeat the process to launch the additional application server required for the internal load balancer configuration.

**App Tier EC2 Instance Screenshot**

<img width="1907" height="905" alt="Screenshot 2026-10-19 002545" src="https://github.com/user-attachments/assets/a7e37d15-cc3d-484c-aa4c-b165e112b41f" />

<img width="1902" height="907" alt="Screenshot 2026-10-19 002654" src="https://github.com/user-attachments/assets/774c1a96-3d77-4a91-91f8-f60134b14f32" />
<img width="1902" height="907" alt="Screenshot 2026-10-19 002654" src="https://github.com/user-attachments/assets/1de6f603-c4a6-4e44-901a-b1d949a93735" />
<img width="1916" height="896" alt="Screenshot 2026-10-19 002713" src="https://github.com/user-attachments/assets/9c2366ab-1939-4d6d-af94-4c50b17b686f" />

---

### Connect to App Instances Using AWS Systems Manager

AWS Systems Manager Session Manager is used to connect to the application-tier EC2 instances without requiring a bastion host or direct inbound SSH access.

1. Open the EC2 console and select the application server instance.
2. Click **Connect** and select the **Session Manager** tab.
3. Click **Connect** to open a browser-based terminal.
4. Verify that the instance has the required IAM permissions, Systems Manager agent configuration, and network connectivity.

Session Manager provides secure administrative access without requiring a publicly accessible IP address or an inbound SSH rule.

**Session Manager Connection Screenshot**

<img width="1890" height="557" alt="Screenshot 2026-10-19 003117" src="https://github.com/user-attachments/assets/14e19b55-7f9f-4f9b-b426-d337f7f8a901" />

---

### Configure Database Connectivity

The application-tier EC2 instance is configured to connect to Amazon RDS for MySQL through the private network.

1. Install the MySQL client on the application server.

   ```bash
   sudo yum install mysql -y
   ```
<img width="1885" height="1002" alt="Screenshot 2026-10-19 003245" src="https://github.com/user-attachments/assets/674aaeea-9083-4686-bf94-2616b4b1042a" />

2. Connect to the RDS MySQL database using its endpoint and database username.

   ```bash
   mysql -h <RDS-ENDPOINT>  -u <DB-USERNAME> -p
   ```

3. Enter the database password when prompted.

4. Verify the database connection and execute the required SQL commands.

The Database Security Group allows inbound MySQL traffic on TCP port **3306** from the App Server Security Group.

**RDS MySQL Connectivity Screenshot**

<img width="1917" height="910" alt="Screenshot 2026-10-19 003317" src="https://github.com/user-attachments/assets/a2a5dea9-1ae9-42b5-a938-faea736466ee" />


### Configure the Database Schema

After connecting to RDS MySQL, create the application database and its required tables if they do not already exist.

Example commands:

```sql
CREATE DATABASE IF NOT EXISTS webappdb;

USE webappdb;

CREATE TABLE IF NOT EXISTS transactions (
    id INT NOT NULL AUTO_INCREMENT,
    amount DECIMAL(10,2),
    description VARCHAR(100),
    PRIMARY KEY (id)
);

INSERT INTO transactions (amount, description)
VALUES (400, 'groceries');

SELECT * FROM transactions;
```

These commands are examples of database initialization. Use the actual schema and data you created for your project, and avoid inserting duplicate test records if you have already populated the table.


### Configure the App Tier

1. Configure the application database settings with the RDS endpoint, database name, username, and password.
2. Upload the application code to the private Amazon S3 bucket.
3. Connect to the application server using Session Manager.
4. Download the application files from the S3 bucket to the EC2 instance.
5. Install the runtime, dependencies, and other components required by the backend application.
6. Start the application and verify that it is running correctly.

The exact runtime and startup commands depend on the application code deployed in the project.

**Application Deployment Screenshot**

<img width="1906" height="921" alt="Screenshot 2026-10-19 003433" src="https://github.com/user-attachments/assets/6a9ea708-aef1-4bb6-92a5-62e44bb01c20" />

<img width="1905" height="961" alt="Screenshot 2026-10-19 003528" src="https://github.com/user-attachments/assets/ab629359-77d2-4229-869c-581c6f9de469" />
<img width="1905" height="961" alt="Screenshot 2026-10-19 003528" src="https://github.com/user-attachments/assets/64da0249-bda7-4d75-843d-14360bd4d773" />
<img width="1903" height="910" alt="Screenshot 2026-10-19 003605" src="https://github.com/user-attachments/assets/1702dc6c-bb20-40e5-a515-27c117ae421e" />
<img width="1886" height="922" alt="Screenshot 2026-10-19 003622" src="https://github.com/user-attachments/assets/83d5b24b-2f92-4c5a-8d9c-8a42752a5299" />
<img width="1917" height="902" alt="Screenshot 2026-10-19 003638" src="https://github.com/user-attachments/assets/d73b777e-eb5d-4631-aeb9-ba09a8e07f98" />

---

### Internal Load Balancer Integration

The application-tier EC2 instances are registered as targets of the internal Application Load Balancer. The internal load balancer distributes incoming requests across the registered targets according to its listener and target group configuration.

Health checks are configured to determine whether the registered targets are healthy and eligible to receive traffic.

The App Server Security Group allows application traffic from the Internal Load Balancer Security Group on the configured application port.

**Internal Load Balancer and Target Group Screenshot**

<img width="1905" height="895" alt="Screenshot 2026-10-19 003708" src="https://github.com/user-attachments/assets/307aaf58-ac49-4799-bfea-609b65a75dbe" />


<img width="1900" height="902" alt="Screenshot 2026-10-19 003716" src="https://github.com/user-attachments/assets/677f1607-7ec1-431d-aae4-85a2568afebe" />


<img width="1901" height="890" alt="Screenshot 2026-10-19 003725" src="https://github.com/user-attachments/assets/cd46f2ca-0a4d-47ca-981d-c0a853346537" />


---

### Application Tier Verification

The application tier was configured manually by launching EC2 instances, installing the required software, retrieving the application code from Amazon S3, and configuring database connectivity.

The instances are integrated with the internal Application Load Balancer to support request distribution across the registered application servers.



### Test App Tier

Now let's run a couple tests to see if our app is configured correctly and can retrieve data from the database.

To hit out health check endpoint, copy this command into your SSM terminal. This is our simple health check endpoint that tells us if the app is simply running.

```shell
curl http://localhost:4000/health
```

The response should looks like the following:

```
"This is the health check"
```

Next, test your database connection. You can do that by hitting the following endpoint locally:

```bash
curl http://localhost:4000/transaction
```

You should see a response containing the test data we added earlier:

```json
{
  "result": [
    { "id": 1, "amount": 400, "description": "groceries" },
    { "id": 2, "amount": 100, "description": "class" },
    { "id": 3, "amount": 200, "description": "other groceries" },
    { "id": 4, "amount": 10, "description": "brownies" }
  ]
}
```

If you see both of these responses, then your networking, security, database and app configurations are correct.

<b>Congrats! Your app layer is fully configured and ready to go. </b>

### Target Group

1. While the AMI is being created, we can go ahead and create our target group to use with the load balancer. On the EC2 dashboard navigate to <b>Target Groups</b> under <b>Load Balancing</b> on the left hand side. Click on <b>Create Target Group</b>.

2. The purpose of forming this target group is to use with our load blancer so it may balance traffic across our private app tier instances. Select Instances as the target type and give it a name.

   Then, set the protocol to <b>HTTP</b> and the port to 4000. Remember that this is the port our Node.ja app is running on. Select the VPC we've been using thus far, and then change the health check path to be <b>/health</b>. This is the health check endpoint of our app. Click <b>Next</b>.

3. We are NOT going to register any targets for now, so just skip that step and create the target group.

### Internal Application Load Balancer

1. In the EC2 console, navigate to **Load Balancers** under **Load Balancing** and click **Create Load Balancer**.
2. Select **Application Load Balancer**.
3. Enter a suitable name for the load balancer and select **Internal** as the scheme, because it routes application requests from the web tier to the app tier.
4. Select the project VPC and the appropriate private subnets in the required Availability Zones.
5. Select the **IALB Security Group**. Ensure that it allows inbound HTTP traffic on port `80` from the **Web Server Security Group**.
6. Configure an HTTP listener on port `80` and set its default forwarding action to the app-tier target group created earlier. The target group forwards requests to the application servers on port `4000`.
7. Create the load balancer and wait until it becomes active.
8. Register the app-tier EC2 instances with the target group if they have not already been registered. Verify that the instances become healthy before testing the application through the internal load balancer.

**Internal Load Balancer Configuration Screenshots this part**


   <img width="1901" height="900" alt="Screenshot 2026-10-19 003754" src="https://github.com/user-attachments/assets/03f9d819-a8c9-4877-80f4-02aceb4ace6b" />


   Select the correct network configuration for VPC and private subnets.


   <img width="1916" height="908" alt="Screenshot 2026-10-19 003809" src="https://github.com/user-attachments/assets/ec31f43b-d97b-403e-93af-4bd81cb30dfc" />


   Select the security group we created for this internal ALB. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to our <b>target group </b>that we just created, so select it from the dropdown, and create the load balancer.


   <img width="1901" height="892" alt="Screenshot 2026-10-19 003830" src="https://github.com/user-attachments/assets/a0941613-b052-4365-95ea-2bd3ebc07f64" />



## Web Tier Instance Deployment - Part 5

In this section we will deploy an EC2 instance for the web tier and make all necessary software configurations for the NGINX web server and React.js website.

- Learning Objectives
  - Update NGINX Configuration Files
  - Create Web Tier Instance
  - Configure Software Stack

### Update Config File

Before we create and configure the web instances, open up the **application-code/nginx.conf** file from the repo we downloaded. Scroll down to **line 58** and replace `[INTERNAL-LOADBALANCER-DNS]` with your internal load balancer’s DNS entry. You can find this by navigating to your internal load balancer's details page.


<img width="1893" height="992" alt="Screenshot 2026-10-19 003928" src="https://github.com/user-attachments/assets/de7fc58c-4ce4-4ed6-baf3-cb45bf1ba1f3" />


<img width="1886" height="982" alt="Screenshot 2026-10-19 003939" src="https://github.com/user-attachments/assets/6b68bfbe-c8c4-4435-9d03-9caf6690b25b" />

Then, upload this file and the **application-code/web-tier** folder to the s3 bucket you created for this project.

### Web Instance Deployment

1. Follow the same instance creation instructions we used for the App Tier instance in **Part 3: App Tier Instance Deployment**, with the exception of the subnet. We will be provisioning this instance in one of our **public subnets**. Make sure to select the correct network components, security group, and IAM role. **This time, auto-assign a public ip** on the **Configure Instance Details page**. Remember to tag the instance with a name so we can identify it more easily.

  <img width="1893" height="902" alt="Screenshot 2026-10-19 003959" src="https://github.com/user-attachments/assets/d141841b-8677-46b8-9e64-31df11c9a882" />


 <img width="1907" height="900" alt="Screenshot 2026-10-19 004017" src="https://github.com/user-attachments/assets/bf233774-ad54-4aeb-adc6-71c387c6bee8" />

   Then at the end, proceed without a key pair for this instance.

### Connect to Instance


<img width="1917" height="901" alt="Screenshot 2026-10-19 004056" src="https://github.com/user-attachments/assets/28a48d47-eac9-401b-b364-81ee896d2869" />

### Configure Web Instance

1. We now need to install all of the necessary components needed to run our front-end application. Again, start by installing NVM and node

   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   nvm install 16
   nvm use 16
   ```

2. Now we need to download our web tier code from our s3 bucket:

   ```bash
   cd ~/
   aws s3 cp s3://BUCKET_NAME/web-tier/ web-tier --recursive
   ```

   Navigate to the web-layer folder and create the build folder for the react app so we can serve our code:

   ```bash
   cd ~/web-tier
   npm install
   npm run build
   ```

3. NGINX can be used for different use cases like load balancing, content caching etc, but we will be using it as a web server that we will configure to serve our application on port 80, as well as help direct our API calls to the internal load balancer.
   ```bash
   sudo amazon-linux-extras install nginx1 -y
   ```
4. We will now have to configure NGINX. Navigate to the Nginx configuration file with the following commands and list the files in the directory:

   ```bash
   cd /etc/nginx
   ls
   ```

   You should see an nginx.conf file. We’re going to delete this file and use the one we uploaded to s3. Replace the bucket name in the command below with the one you created for this project:

   ```bash
   sudo rm nginx.conf
   sudo aws s3 cp s3://BUCKET_NAME/nginx.conf .
   ```

   Then, restart Nginx with the following command:

   ```bash
   sudo service nginx restart
   ```

   To make sure Nginx has permission to access our files execute this command:

   ```bash
   chmod -R 755 /home/ec2-user
   ```

   And then to make sure the service starts on boot, run this command:

   ```bash
   sudo chkconfig nginx on
   ```

5. Now when you plug in the public IP of your web tier instance, you should see your website, which you can find on the Instance details page on the EC2 dashboard. If you have the database connected and working correctly, then you will also see the database working. You’ll be able to add data. Careful with the delete button, that will clear all the entries in your database.

<img width="1915" height="955" alt="Screenshot 2026-10-19 004110" src="https://github.com/user-attachments/assets/99cc2643-2ec4-4c6b-81d0-2844c24b15c5" />

<img width="1911" height="956" alt="Screenshot 2026-10-19 004126" src="https://github.com/user-attachments/assets/b2628fc6-afff-4268-b805-d21f6aecf88f" />
<img width="1892" height="906" alt="Screenshot 2026-10-19 004205" src="https://github.com/user-attachments/assets/f30c7cbc-e526-4f6a-9f7f-e5e363aad2f7" />
<img width="1910" height="961" alt="Screenshot 2026-10-19 004406" src="https://github.com/user-attachments/assets/78543b2a-aa9b-4b01-b24b-6c41d270c323" />
<img width="1910" height="926" alt="Screenshot 2026-10-19 004426" src="https://github.com/user-attachments/assets/bbfa8ec9-cb37-4be2-bc77-67aa011e4793" />
<img width="1910" height="908" alt="Screenshot 2026-10-19 004512" src="https://github.com/user-attachments/assets/d582f38e-8ded-4929-a36b-0bc67c161524" />

## External Load Balancer Deployment - Part 6

In this section, we will configure an internet-facing Application Load Balancer (EALB) to route incoming client requests to the web-tier EC2 instance or instances. The load balancer uses a target group and health checks to determine which registered web servers are eligible to receive traffic. The web-tier EC2 instances are launched and configured manually; Auto Scaling is not implemented in this project.

- Learning Objectives:
  - Create a target group for the web tier.
  - Configure an internet-facing Application Load Balancer.
  - Configure listeners and forwarding rules.
  - Verify web-tier target health and application access.


### Target Group

1. Open the EC2 console and navigate to **Target Groups** under **Load Balancing** on the left-hand side. Click **Create target group**.

2. Select **Instances** as the target type and provide a suitable name for the target group.

3. Set the protocol to **HTTP** and the port to `80`, because Nginx listens on port 80 on the web-tier EC2 instance. Select the VPC used for this project.

4. Configure the health check path to `/` if it returns a successful HTTP response from your Nginx web server. Otherwise, use a path that returns a successful response.

5. Register your web-tier EC2 instance or instances with the target group, then create the target group.

### Internet Facing Load Balancer

1. On the left hand side of the EC2 dashboard select **Load Balancers** under **Load Balancing** and click **Create Load Balancer.**

2. We'll be using an **Application Load Balancer** for our HTTP traffic so click the create button for that option.

3. After giving the load balancer a name, be sure to select **internet facing** since this one will not be public facing, but rather it will route traffic from our web tier to the app tier.

   Select the correct network configuration for VPC and **public** subnets.

Select the **EALB Security Group** created for the internet-facing Application Load Balancer. Configure the listener on HTTP port 80, or HTTPS port 443 if TLS is configured. Select the web-tier target group as the default forwarding destination.

Ensure that the EALB Security Group allows inbound traffic from the intended client IP ranges and outbound traffic to the web-tier targets on the required listener and health-check ports. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to our **target group** that we just created, so select it from the dropdown, and create the load balancer.

<img width="1915" height="920" alt="Screenshot 2026-10-19 005051" src="https://github.com/user-attachments/assets/f5b78c6f-958e-4ae2-9087-9d678dc985f4" />


### ** Congrats! You’ve Implemented a 3 Tier Web Architecture! **<br>

## Clean Up

In order to clean up the resources in the AWS account and avoid incurring any other charges, you'll want to delete the resources in a specific order. We will start with the different services deployed inside the VPC, and then the VPC components themselves:


1. **Application Load Balancers**

   For each of the Application Load Balancers we created, start by deleting the listeners. Then, delete the load balancers by checking the box next to them, and then clicking delete under the Actions dropdown.

3. **Target Groups**

   Now that there's no Application Load Balancer dependency on target groups we can delete them. Navigate to Target Groups on the left hand side of the Ec2 dashboard. Select your target groups and click delete under the Actions dropdown.


4. ** EC2 Instances**

    Navigate to Instances under Instances in the left hand side of the EC2 dashboard. Select all instances and click Terminate Instance under the Instance State dropdown.

7. **RDS MySQL Database**

   To delete the database we created, navigate to the RDS service dashboard. We need to make sure that deletion protection is off. Do this by selecting the regional cluster and clicking the **Modify** button. Scroll all the way down and you should be able to uncheck deletion protection if it is checked.

   On the next screen, select **Apply Immidiately** and then click **Modify Cluster**.

   Now we can delete the reader and writer nodes separately. To do that, select one of them and click **delete** from the Actions dropdown. You'll have to type **delete me** into the resulting pop-up. Repeat for the other database instance. For the last instance the pop up with ask to create a snapshot. Don't do this and check off the acknowledgement.

12. **VPC**

    Lastly, delete the VPC by navigating to Your VPCs on the lefthand side of the VPC dashboard. Select the VPC that we created for this lab, and under the Actions dropdown click Delete VPC. This will also delete the subnets and security groups.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
