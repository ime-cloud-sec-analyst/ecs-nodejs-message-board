# ecs-nodejs-message-board
Containerized Node.js message board application refactored into microservices and deployed on AWS ECS with an Application Load Balancer.
ChatGPT said:
ECS Node.js Message Board Lab
Containerized Node.js message board application refactored into microservices and deployed on AWS ECS with an Application Load Balancer

Lab Overview
In this guided lab, we migrated a monolithic Node.js message board application into Docker containers, deployed it on AWS ECS (EC2 launch type), and then refactored it into three independent microservices (Users, Threads, Posts). Finally, we configured an ALB to route requests by path.

<p align="center"><img src="images/01_initial-browser.png" alt="Lab start" width="600"></p>
Table of Contents
Prerequisites & Environment Setup

Running the Monolith Locally

Containerizing the Monolith

Deploying the Monolith on ECS

Troubleshooting Health Check Failures

Refactoring into Microservices

Deploying Microservices & Path-based Routing

Lessons Learned

Lab Artifacts

Prerequisites & Environment Setup
AWS Cloud9 IDE with terminal access

AWS CLI & Docker installed in Cloud9

IAM permissions to create ECR repos, ECS clusters, ALBs, Security Groups

Download lab files

bash
Copy
Edit
curl -s https://…/lab-files-ms-node-js.tar.gz | tar -zxv
Folders:

1-no-container (monolith on Node.js)

2-containerized-monolith (Docker-ready monolith)

3-containerized-microservices (split into users, threads, posts)

<p align="center"><img src="images/02_curl-node-modules.png" alt="Installing Node.js modules" width="400"></p>
Running the Monolith Locally
Install dependencies in 1-no-container:

bash
Copy
Edit
npm install koa koa-router
Start the app:

bash
Copy
Edit
npm start
Test APIs (new terminal):

bash
Copy
Edit
curl http://localhost:3000/api/users
curl http://localhost:3000/api/threads
<p align="center"><img src="images/03_curl-users.png" alt="API users local" width="400"></p>
Containerizing the Monolith
Review 2-containerized-monolith/Dockerfile

Base: node:alpine

WORKDIR /srv → copy files → EXPOSE 3000 → CMD node server.js

Create ECR repo mb-repo via console

Authenticate Docker, build, tag, push:

bash
Copy
Edit
aws ecr get-login-password … | docker login …
docker build -t mb-repo .
docker tag mb-repo:latest <ACCOUNT>.dkr.ecr.us-west-2.amazonaws.com/mb-repo:latest
docker push <ACCOUNT>.dkr.ecr.us-west-2.amazonaws.com/mb-repo:latest
<p align="center"><img src="images/04_ecr-push.png" alt="Pushing to ECR" width="600"></p>
Deploying the Monolith on ECS
Create ECS Cluster (EC2, 2 × t2.medium, Amazon Linux 2023)

Task Definition mb-task:

CPU .5 vCPU, Mem 1 GB

Container mb-container image → port 3000

Security Group

Inbound: HTTP 80 (0.0.0.0/0), TCP 3000 restricted to ALB SG

Service mb-task-service:

Launch type EC2, desired 1

Attach to new ALB mb-load-balancer → listener HTTP 80 → target group mb-target on port 300

<p align="center"><img src="images/05_service-creation.png" alt="Service creation" width="600"></p>
Troubleshooting Health Check Failures
Symptom: ECS circuit breaker triggered, tasks stopped as “port 300 is unhealthy → Request timed out.”

Cause: Health check configured for container port 300, but the app listens on 3000.

Fix:

Update task definition port mapping to container port 3000 (leave host port blank).

Create new revision, update service to use revision 2.

Redeploy → tasks become healthy.

<p align="center"><img src="images/06_port-mapping-fix.png" alt="Port mapping correction" width="600"></p>
Refactoring into Microservices
Split monolith into three folders under 3-containerized-microservices:

users, threads, posts

server.js in each handles only its API subset (/api/users*, /api/threads*, /api/posts*).

Create ECR repos: mb-users-repo, mb-threads-repo, mb-posts-repo

Build & push images for each microservice (repeat Docker push flow).

<p align="center"><img src="images/07_microservices-folders.png" alt="Microservices folders" width="600"></p>
Deploying Microservices & Path-based Routing
Task Definitions:

mb-users-task, mb-posts-task, mb-threads-task

Container port 3000

Services: mb-users-service (path /api/users*), mb-posts-service (/api/posts*), mb-threads-service (/api/threads*)

ALB Listener Rules (HTTP 80):

/api/users* → mb-users-target

/api/posts* → mb-posts-target

/api/threads* → mb-threads-target

Default → mb-users-target (serves root or invalid)

<p align="center"><img src="images/08_listener-rules.png" alt="ALB listener rules" width="600"></p>
Lessons Learned
Health Checks & Ports: Always align container’s listening port with ECS task definitions and ALB health checks.

ECS Revisions: Increment task definition revision for every port or resource change, then update service.

Security Groups: Use SG references (allow ALB SG → container SG), avoid wide-open port ranges.

Microservices Routing: ALB path-based routing lets you route multiple services through one load balancer.

Incremental Debugging: Deploy monolith first, verify, then refactor; makes isolating issues easier.

Lab Artifacts
All terminal outputs, screenshots, and architecture diagrams are in the images/ directory, numbered in execution order:

pgsql
Copy
Edit
images/
├─ 01_initial-browser.png
├─ 02_curl-node-modules.png
├─ 03_curl-users.png
├─ 04_ecr-push.png
├─ 05_service-creation.png
├─ 06_port-mapping-fix.png
├─ 07_microservices-folders.png
├─ 08_listener-rules.png
└─ …etc. (up to 40)
