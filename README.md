# Project: Designing Adaptive Auto-Scaling Architecture for High Availability and Cost Optimization

**GitHub Repository**  
https://github.com/DinukaSanjana/Designing-Adaptive-Auto-Scaling-Architecture-for-HA-and-cost-optimisat.git

## How High Availability and Cost Optimization Are Achieved

| Goal                  | How It Is Achieved                                                                 |
|-----------------------|-------------------------------------------------------------------------------------|
| **High Availability** | • Multi-AZ deployment  <br>• Application Load Balancer distributes traffic  <br>• Auto Scaling Group maintains desired number of healthy instances  <br>• If an instance fails or is terminated, ASG automatically launches a new one and ALB routes traffic only to healthy instances |
| **Cost Optimization** | • Auto Scaling Group scales out only when traffic increases  <br>• Scales in automatically when traffic decreases  <br>• Minimum 2 instances (for HA), but never runs more than needed  <br>• Pay only for the instances actually required |

**Traffic Flow**  
`User Traffic → Application Load Balancer → Target Group → Healthy EC2 Instances (Docker containers)`  
Auto Scaling Group continuously monitors instance health and metrics, adding or removing instances as needed.

## Technologies Used
- EC2 – Compute instances
- Docker – Container runtime
- AMI – Golden image with app + Docker pre-installed
- ALB (Application Load Balancer) – Distributes incoming traffic
- ASG (Auto Scaling Group) – Maintains desired count and auto-scales
- RDS (MySQL) – Centralized managed database

## Detailed Implementation Steps

### 1. Create RDS MySQL Database
- Launch RDS MySQL instance
- Copy the Endpoint
- Edit `app.js` → “Create connection to MySQL” section → hardcode RDS endpoint, username, password, database

### 2. Add Inbound Rule to RDS Security Group
- Type: MySQL/Aurora (port 3306)
- Source: Security Group of the EC2 instances (recommended)  
  → Allows EC2 instances to connect to RDS

### 3. Create EC2 Instance (Golden Instance)
- AMI: Ubuntu
- Type: t2.medium
- Security Group Inbound Rules:
  - SSH (22) – My IP
  - HTTP (80) – Anywhere
  - HTTPS (443) – Anywhere
  - Custom TCP 3000 – Anywhere (0.0.0.0/0)

**Why open port 3000 to Anywhere?**  
Because initially we test the app directly on the instance (`PublicIP:3000`). Later ALB will use the same port, and the same SG is reused.

### 4. Connect to EC2 Instance & Run Application
```bash
# Install Docker and post-install steps
sudo apt update && sudo apt install docker.io -y
sudo usermod -aG docker ubuntu

# Clone repository
git clone https://github.com/MuhmmadAyan/crud-app.git
cd crud-app
cp -r * ../   # if needed to move files

# Build Docker image
docker build -t crud-image .

# Run container (auto-restart on reboot)
docker run --restart=always -d -p 3000:3000 crud-image

# Verify container is running
docker ps
```

Test in browser: `http://<EC2-Public-IP>:3000`

### 5. Fix RDS Connection (if needed)
```bash
# Stop old container
docker stop <container-id>

# Run again with same command
docker run --restart=always -d -p 3000:3000 crud-image
```

### 6. Create AMI (Golden Image)
EC2 Console → Select running instance → Actions → Image and templates → Create image  
- Image name: `crud-ami`  
- Uncheck “No reboot” if you want faster creation (optional)

### 7. Create Application Load Balancer & Target Group
- Load Balancer → Create ALB
- Name: `crud-elb`
- Listener: HTTP :80
- Create Target Group:
  - Name: `crud-tg`
  - Protocol: HTTP port 3000
  - Health check interval: 6s
- Register the current instance (temporary)
- Create Load Balancer

Now application is accessible via ALB DNS name (no port needed)

### 8. Create Launch Template
- Name: `crud-launch-template`
- AMI: `crud-ami` (created in step 6)
- Instance type: t2.medium
- Same Security Group as before
- Advanced → User Data:
```bash
#!/bin/bash
docker run --restart=always -d -p 3000:3000 crud-image
```
**Purpose of User Data:** When ASG launches new instances from the AMI, the Docker container is not running by default. This script automatically starts the container on every new instance boot.

### 9. Create Auto Scaling Group
EC2 → Auto Scaling Groups → Create Auto Scaling Group
- Name: `crud-asg`
- Launch template: `crud-launch-template`
- VPC + multiple Availability Zones
- Attach to existing load balancer → Target Group: `crud-tg`
- Health checks: Enable ELB health checks + Grace period 30s
- Desired capacity: 3
- Minimum: 2
- Maximum: 5
- Scaling policy: Target tracking → Average Network Out → Target value: 50

### 10. Monitoring & Verification
- Auto Scaling Group → Activity tab  
  Shows instance launches, terminations, health status
- EC2 → Instances  
  Shows 3 running instances launched by ASG

### 11. Disaster Recovery / Self-Healing Test
- Manually terminate 1 or all 3 instances
- ASG detects unhealthy/missing instances
- Automatically launches new instances from `crud-ami` + runs User Data script
- New instances register to Target Group → become healthy
- ALB resumes routing traffic → Application stays available

**Result:** Zero-downtime (or <1 minute) even when instances are deleted.

## Final Architecture
```
Internet
   ↓
Application Load Balancer (HTTP:80)
   ↓
Target Group (HTTP:3000 + Health Checks)
   ↓
Auto Scaling Group (Min 2, Desired 3, Max 5)
   ↓
EC2 Instances (Docker → Node.js CRUD App)
   ↓
Amazon RDS MySQL (shared database)
```

**Project Successfully Demonstrates**
- High Availability across multiple AZs
- Automatic scaling based on load
- Self-healing on failure
- Cost optimization by scaling in/out
- Fully reproducible infrastructure using AMI + Launch Template + User Data

**Completed**
```
