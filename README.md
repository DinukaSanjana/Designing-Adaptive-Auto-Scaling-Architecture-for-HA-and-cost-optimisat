# Project-8: Designing Adaptive Auto-Scaling Architecture for HA and Cost Optimization

## GitHub Repo
https://github.com/DinukaSanjana/Designing-Adaptive-Auto-Scaling-Architecture-for-HA-and-cost-optimisat.git

## Project Purposes
1. High Availability (HA): Ensure the application is always available to users. If one instance fails, others automatically take over.
2. Cost Optimization: Avoid unnecessary costs by automatically reducing instances when traffic is low and increasing when traffic is high.

## Technologies Used
- Amazon EC2 (Elastic Compute Cloud): Virtual servers for the application.
- Docker: Containerize the Node.js application with dependencies (Node 14, express, mysql).
- Amazon Machine Image (AMI): Snapshot of configured EC2 instance for auto scaling.
- Application Load Balancer (ALB): Distribute user traffic across EC2 instances.
- AWS Auto Scaling Group (ASG): Automatically scale EC2 instances based on CPU, network traffic.
- Amazon RDS (Relational Database Service): Managed MySQL database connected to all instances.

## Lesson-by-Lesson Workflow

### Lesson 1: Introduction
Introduce HA and Cost Optimization architecture. Deploy EC2 instances in Multi-AZ within VPC, fronted by ALB and managed by ASG.

### Lesson 2: App Explanation
The application is a simple Node.js CRUD app using express and mysql dependencies, running on port 3000, connected to external AWS RDS database. Dockerfile provided for deployment.

### Lesson 3: EC2 App Setup
1. Launch Instance: Ubuntu EC2 instance.
2. Security Group: Open ports 3000, SSH, HTTP, HTTPS.
3. Install Docker: On EC2 instance.
4. Get App: Clone source code to instance.
5. Build & Run: Use Dockerfile to build image, run container on port 3000.
6. Test: Access via public IP and port 3000.

### Lesson 4: AMI and ELB Setup
1. Create AMI: From configured EC2 instance, name "Crud-ami".
2. Create Target Group (TG): Name "Crud-TG", target port 3000, with health checks.
3. Create Load Balancer (ALB): Name "Crud-ELB", internet-facing, listen on port 80, forward to TG.

### Lesson 5: ASG Setup
1. Create Launch Template: Name "Crud Launch template".
   - AMI: Use "Crud-ami".
   - Instance Type: t2.medium.
   - User Data: Script to run docker container on boot.
2. Create Auto Scaling Group (ASG): Name "Crud-ASG".
   - Use Launch Template.
   - Attach to "Crud-TG".
   - Group Size: Min 2, Desired 3, Max 5.
   - Scaling Policy: Scale out if CPU >50% or high network out.
3. Test: ASG launches 3 instances, registers to TG, access via ALB DNS.

### Lesson 6: Disaster Simulation
1. Simulate Disaster: Manually terminate instances.
2. Automatic Recovery:
   - ALB detects unhealthy instances.
   - ASG launches new instances to maintain desired capacity.
   - New instances register as healthy and handle traffic.
3. Conclusion: System self-heals with minimal downtime.

## Services Used
- EC2
- ASG
- DB (RDS)
- LoadBalancer

## Create RDS - MySQL Database
Copy the Endpoint from Amazon RDS > Databases > testdb-1 > Connectivity & security.

Hardcode endpoint URL to app.js in "Create connection to MySQL section".

## Add Dbeaver IP to Database Security Group
Add inbound rule for database security group with computer's IP.

## Create EC2 Instance
- Name: Crud-app
- OS: ubuntu
- Type: t2.medium
- Key pair: Proceed without
- Network Settings:
  - Firewall (security groups): Create security group
  - Allow SSH from anywhere
  - Allow HTTPS from internet
  - Allow HTTP from internet
- Inbound Security Group Rules:
  - Add rule: Custom TCP, Port 3000, Source Anywhere
- Configure Storage: 1 x 12 GiB
- Launch Instance

## Connect to Instance
Install Docker inside instance.

Install "Linux post-installation steps for Docker Engine".

Close and reconnect to instance.

## Commands in Instance
whoami  
ubuntu  

Clone the repo:  
git clone https://github.com/MuhmmadAyan/crud-app.git  

cd crud-app  
cp -r * ../  

docker build -t crud-image .  
docker images  

docker run --restart=always -d -p 3000:3000 crud-image  
docker ps  

Copy public IP and paste on browser with port: 13.234.32.214:3000

## AMI (Amazon Machine Image) and ELB (Elastic Load Balancer)
Instance > crud-app > Actions > Image and Template > Create image  
- crud-ami  
- crud-ami  
- [ ] Reboot Instance (Uncheck)  
Create Image  

Image shows in EC2 > AMIs

## Creating the Load Balancer
Go to EC2 > Load Balancers > Create Load Balancer  
Application Load Balancer > Create  

Load Balancer Name: crud-elb  
VPC: Default VPC  
Availability Zones: Select all  

Select security group from EC2 instance (allows port 3000)  

Listeners and Routing:  
HTTP 80  

[Create Target Group]  
Target group name: crud-tg  
Protocol: HTTP > 3000  
Advanced health check settings: Interval 6s  

Next  
Register targets: Available instances > Select crud-app instance  
[Include as pending below]  
[Create Target Group]  

Refresh and select created Target Group  
[Create Load Balancer]

## ASG Setup - Auto Scaling Group
Open application through Load Balancer DNS Name: crud-elb-577084204.us-east-1.elb.amazonaws.com  

EC2 > Auto Scaling Group > Create Auto Scaling group  
crud-ASG  

Create Launch Template  
crud-launch-template  
crud-launch-template  

Application and OS Images: My AMIs > Own by me > Select "crud-ami"  
Instance type: t2.medium  
Key Pair: KEY_NAME  
Select previous security group  

Advanced details: User data - optional  
#!/bin/bash  
docker run --restart=always -d -p 3000:3000 crud-image  

[Create launch template]  

Launch Template: Select created one  
Next  

Network: Select 3 availability zones  
Next  

Load Balancing: Attach to existing load balancer  
Choose from load balancer target group: crud-TG  
Health Checks: Turn on Elastic Load Balancing health checks  
Health grace period: 30s  
Next  

Configure group size and scaling:  
Desired capacity: 3  
Min: 2  
Max: 5  
Automatic scaling: Target tracking scaling policy  
Metric Type: Average network out (byte)  
Target value: 50  
Instance warmup: 30  
Next  

Add notifications - optional  
Next  

Add Tags [optional]  
Review  
[Create Auto Scaling group]  

Auto scaling group > activity  
EC2 > Instances  

## Disaster Simulation
Delete all instances.  
They recreate automatically.  
Auto Scaling Group > Activity shows instances.  

Finished.
