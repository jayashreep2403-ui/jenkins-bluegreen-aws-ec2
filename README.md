---Automated CI/CD Pipeline with Jenkins & Blue-Green Deployment on AWS----

Overview
This project implements a fully automated CI/CD pipeline using Jenkins and GitHub, deploying a web application to AWS EC2 using a Blue-Green deployment strategy via AWS CodeDeploy and an Application Load Balancer (ALB).
The goal was to eliminate manual deployments and reduce downtime during releases. With this pipeline, every push to the main branch automatically triggers a zero-downtime Blue-Green deployment — with instant rollback capability if the new version fails health checks.

Architecture
Developer pushes code
        │
        ▼
   GitHub (main branch)
        │  webhook (POST trigger)
        ▼
     Jenkins
        │  build → test → deploy
        ▼
  AWS CodeDeploy
        │
        ▼
Application Load Balancer (ALB)
       / \
      /   \
     ▼     ▼
  Blue     Green
  EC2s     EC2s
(standby) (new live)
Flow:

Code pushed to GitHub → webhook fires → Jenkins pipeline starts
Jenkins builds the app and runs tests
Jenkins triggers AWS CodeDeploy
CodeDeploy deploys new version to Green EC2 instances
ALB health checks pass → traffic switches from Blue → Green
Blue becomes standby (instant rollback target)


Tech Stack
ToolPurposeJenkinsCI/CD orchestration, pipeline executionGitHubSource control + webhook triggerAWS CodeDeployDeployment automation + Blue-Green switchingAWS EC2Application hosting (Blue & Green environments)AWS ALBTraffic routing between Blue and GreenAWS IAMLeast-privilege permissions for Jenkins & CodeDeployAWS S3Artifact storage for deployment bundlesAWS SNSDeployment failure notifications

Repository Structure
├── Jenkinsfile                  # Pipeline definition (Build → Test → Deploy stages)
├── appspec.yml                  # CodeDeploy lifecycle hooks configuration
├── index.html                   # Application entry point
└── scripts/
    ├── install_dependencies.sh  # Runs on EC2 before deployment (BeforeInstall)
    ├── start_server.sh          # Starts the application (ApplicationStart)
    └── stop_server.sh           # Gracefully stops old version (BeforeInstall)

Key Files Explained
Jenkinsfile
Defines 3 pipeline stages:

Build — packages application artifacts
Test — runs health/smoke tests
Deploy — invokes AWS CodeDeploy to trigger Blue-Green deployment

appspec.yml
CodeDeploy configuration file that maps lifecycle hooks:
yamlversion: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies.sh
  ApplicationStart:
    - location: scripts/start_server.sh
  ApplicationStop:
    - location: scripts/stop_server.sh
scripts/install_dependencies.sh
Installs Apache and CodeDeploy agent on the target EC2. Also ensures codedeploy-agent is enabled on reboot via systemctl enable.

Blue-Green Deployment Strategy
AspectDetailWhy Blue-Green?Keeps two identical environments — zero risk during releaseTraffic switchALB listener rules updated by CodeDeploy — instant cutoverRollback time< 2 minutes — just point ALB back to Bluevs RollingRolling has mixed-version window; Blue-Green never doesDowntimeZero — ALB switches only after Green passes health checks

IAM Setup (Least Privilege)
Two IAM roles created:
Jenkins IAM User — permissions:

codedeploy:CreateDeployment
codedeploy:GetDeployment
s3:PutObject (artifact upload)
ec2:DescribeInstances

CodeDeploy Service Role — permissions:

AWSCodeDeployRole managed policy
EC2 describe + tag access


Jenkins Configuration

Plugin: GitHub plugin + AWS CodeDeploy plugin
Credentials: AWS keys stored in Jenkins Credentials Manager (never hardcoded)
Trigger: GitHub webhook on push to main branch
Notifications: SNS alert on deployment failure


Problems Faced & How I Fixed Them
Problem 1: CodeDeploy agent not running after EC2 reboot

Deployments were silently failing on restarted instances
Fix: Added systemctl enable codedeploy-agent to install_dependencies.sh so agent starts on every boot

Problem 2: Jenkins webhook not triggering on push

GitHub webhook was returning 302 redirect
Fix: Corrected Jenkins URL in webhook settings to include /github-webhook/ trailing slash

Problem 3: ALB health checks failing on Green environment

Apache wasn't starting fast enough before health check hit
Fix: Added a 30-second sleep in start_server.sh and increased ALB health check grace period


Results
MetricBeforeAfterDeployment time30-45 min (manual)~5 min (automated)Downtime per release10-15 minutes0 minutesRollback time1-2 hours< 2 minutesHuman error riskHighEliminated

How to Run This Project
Prerequisites

AWS account with EC2, CodeDeploy, ALB, S3, IAM access
Jenkins server running on EC2 (port 8080)
GitHub repository with webhook configured

Steps

Fork this repo and configure your GitHub webhook to point to http://<jenkins-ec2-ip>:8080/github-webhook/
Create two EC2 Auto Scaling Groups — one for Blue, one for Green — registered to separate ALB target groups
Create CodeDeploy Application:

   Platform: EC2/On-premises
   Deployment Type: Blue/Green
   Load Balancer: Your ALB
   Deployment Config: CodeDeployDefault.OneAtATime

Configure Jenkins:

Install GitHub + AWS CodeDeploy plugins
Add AWS credentials to Jenkins Credentials Manager
Create Pipeline job pointing to this repo's Jenkinsfile


Push to main — pipeline triggers automatically


Lessons Learned

Blue-Green is more expensive (double EC2 cost) but worth it for production workloads where downtime = lost revenue
Always enable codedeploy-agent on boot — not just install it
Least-privilege IAM from day one saves debugging time later
Jenkins webhook failures are almost always a trailing slash issue


Connect
If you're working on similar DevOps projects or have questions about this implementation, feel free to reach out!
