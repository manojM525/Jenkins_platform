Jenkins CI Platform Runbook

(Architecture, Setup, Operations, and Recovery Guide)

1. System Overview

This platform implements a containerized Jenkins CI system designed for:

reproducibility

scalability

disaster recovery

infrastructure as code

The system builds application images and pushes them to Amazon ECR, which are then deployed to Amazon EKS.

2. High-Level Architecture
Developer
   │
   │ push
   ▼
GitHub Repository
   │
   │ webhook
   ▼
Jenkins Controller (Docker on EC2)
   │
   │ schedules pipelines
   ▼
Ephemeral Docker Agent Containers
   │
   │ build application
   ▼
Docker Image
   │
   │ push
   ▼
Amazon ECR
   │
   │ deployment artifact
   ▼
Amazon EKS
3. Infrastructure Layout
AWS EC2 Instance (t3.large)

│
├ Root EBS Volume
│    OS + Docker
│
├ Jenkins Controller Container
│
├ Docker Engine
│
├ Docker Network
│    jenkins-net
│
├ Ephemeral Jenkins Agent Containers
│
└ Dedicated Jenkins EBS Volume
     mounted at
     /data/jenkins
         ↓
     mapped inside container
         ↓
     /var/jenkins_home
4. Key Architectural Decisions
4.1 Jenkins Runs in Docker

Controller runs as a container instead of being installed directly.

Benefits:

reproducible CI infrastructure

easy upgrades

simple rollback

container isolation

Controller image is built from:

jenkins/jenkins:lts
4.2 Plugins Installed During Image Build

Plugins are installed using:

plugins.txt

Installed during Docker image build.

Advantages:

reproducible plugin versions

faster Jenkins startup

no runtime plugin downloads

enables vulnerability scanning

4.3 Dedicated EBS Volume for Jenkins Data

Mounted path:

/data/jenkins

Mapped inside container:

/var/jenkins_home

This directory stores:

jobs
plugins
workspace
logs
build history
credentials

Benefits:

Jenkins survives EC2 replacement

easy EBS snapshots

easier disaster recovery

clean separation from OS disk

4.4 Docker Network for Jenkins Components

Docker network created:

docker network create jenkins-net

Purpose:

container DNS resolution

agents can reach controller by hostname

avoids localhost networking problems

4.5 Ephemeral Docker Agents

Agents are started dynamically per pipeline.

Benefits:

build isolation

consistent build environments

easier scaling

avoids dependency conflicts

4.6 Language-Specific Agent Images

Instead of one universal agent.

Examples:

node-agent
java-agent
python-agent

Advantages:

smaller images

faster startup

fewer vulnerabilities

reduced attack surface

4.7 Docker Socket Mount

Mounted into Jenkins container:

/var/run/docker.sock

Purpose:

Allows Jenkins pipelines to run:

docker build
docker push
docker run

Trade-off:

less secure
but simpler and stable
4.8 ECR for Container Image Storage

Pipeline output images are stored in:

Amazon ECR

Benefits:

AWS native integration

secure registry

integrates with EKS deployments

5. Storage Architecture
EC2 Instance

│
├ Root EBS
│    /
│
└ Jenkins EBS
      /data/jenkins

Jenkins container mapping:

/data/jenkins → /var/jenkins_home
6. Jenkins Platform Repository Structure
jenkins-platform/

├ controller
│   ├ Dockerfile
│   ├ plugins.txt
│   └ .dockerignore
│
├ agents
│   ├ node-agent
│   ├ java-agent
│   └ python-agent
│
└ jcasc
    └ jenkins.yaml
7. Jenkins Controller Dockerfile

Example:

FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    apt-get install -y git curl jq && \
    apt-get clean

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt

RUN jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt

USER jenkins
8. Running Jenkins Controller
docker run -d \
  --name jenkins-controller \
  --network jenkins-net \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /data/jenkins:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  company-jenkins-controller
9. Backup Strategy

Primary storage:

EBS volume

Backup methods:

EBS Snapshots
periodic snapshots
S3 Backup

Archive Jenkins data:

tar backup of jenkins_home

Lifecycle policy:

retain recent backups
archive older backups
10. Known Operational Bottlenecks
Docker daemon contention

Multiple pipelines share one Docker daemon.

Workspace growth

Directory:

/var/jenkins_home/workspace

Large dependency directories accumulate.

Docker storage growth

Location:

/var/lib/docker

Requires periodic cleanup.

Example cleanup:

docker system prune
File descriptor limits

CI systems open many files and sockets.

Limit checked using:

ulimit -n

Production systems typically increase this to:

65535
Disk I/O pressure

Heavy operations:

git clone
npm install
maven build
docker build
11. CI Pipeline Storm Protection

Pipeline option:

options {
  disableConcurrentBuilds(abortPrevious: true)
}

Purpose:

Prevent multiple builds for the same branch from running simultaneously.

12. Disaster Recovery Procedure

If Jenkins EC2 fails:

Launch new EC2 instance

Attach Jenkins EBS volume

Mount /data/jenkins

Start Jenkins container

Example:

docker run -d \
  --name jenkins-controller \
  -v /data/jenkins:/var/jenkins_home \
  company-jenkins-controller

Jenkins resumes with previous state.

13. Jenkins Upgrade Procedure

Update plugins.txt or Dockerfile

Build new Jenkins image

Stop old container

Start new container with same volume

Example:

docker stop jenkins-controller
docker run new-image

Rollback possible by starting previous image.

14. Monitoring Considerations

Important metrics:

CPU usage
memory usage
disk usage
docker storage
jenkins queue length

Tools typically used:

Prometheus
Grafana
CloudWatch
15. Key References

Jenkins Documentation

https://www.jenkins.io/doc/

Jenkins Docker Image

https://hub.docker.com/r/jenkins/jenkins

Docker Documentation

https://docs.docker.com/

AWS EBS

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEBS.html

AWS ECR

https://docs.aws.amazon.com/AmazonECR/latest/userguide/

16. Current System Status

Current setup:

Docker installed on EC2
Version: 28.2.2

EBS volume attached
Device: /dev/nvme1n1

Mounted at:
/data/jenkins

Ownership set to UID 1000
17. Next Implementation Step

Next step:

Build Jenkins controller image
Run Jenkins container
Access Jenkins UI
Configure admin user
Tip for Continuing in the New Chat

Paste this runbook and write:

Continue Jenkins platform setup from the point where the EBS volume
is mounted at /data/jenkins and Docker is installed on the EC2.
Next step: build and run the Jenkins controller container.

The conversation will resume smoothly.

If you want, in the new chat we can also build a production-grade DevOps portfolio diagram for this system that looks like something a real platform team would present in an architecture review.