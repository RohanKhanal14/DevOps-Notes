# Day 18 - Jenkins Master-Slave Architecture with Docker

This repository demonstrates a local Jenkins setup using a controller/agent architecture. It is designed for learning how Jenkins works in a master-slave model, how agents execute jobs remotely, and how to troubleshoot SSH-based agent connection issues.

---

## What this project demonstrates

In Jenkins, the controller (master) manages jobs, users, credentials, plugins, build history, and artifacts. Agents are remote nodes that actually execute the build steps.

This setup uses:

- A Jenkins controller container
- One Jenkins agent container
- SSH-based agent connection
- Docker access from the agent for build-related tasks

This is a practical way to understand:

- Controller vs. agent responsibilities
- SSH authentication for agents
- Jenkins node configuration
- Build execution isolation and scaling

---

## Repository structure

https://github.com/RohanKhanal14/jenkins-controller-agent-architecture.git
https://github.com/RohanKhanal14/three-tier-crud-app

```bash
.
├── docker-compose.yml
├── controller/
│   └── Dockerfile
├── agent/
│   └── Dockerfile
└── ssh/
    ├── jenkins_agent_key
    └── jenkins_agent_key.pub
```

---

## Prerequisites

Make sure the following are installed on your machine:

- Docker Engine
- Docker Compose plugin

Verify installation:

```bash
docker --version
docker compose version
```

---

## Step 1 - Clone and enter the project

```bash
git clone <your-repo-url>
cd jenkins-docker-local
```

---

## Step 2 - Build and start the containers

Run:

```bash
docker compose up --build -d
```

This will start:

- Jenkins controller on port 8080
- Jenkins agent on port 22 inside the Docker network

Check container status:

```bash
docker compose ps
```

---

## Step 3 - Access Jenkins

Open your browser and go to:

```text
http://localhost:8080
```

Unlock Jenkins using the initial admin password:

```bash
docker exec jenkins-controller cat /var/jenkins_home/secrets/initialAdminPassword
```

Follow the setup wizard:

- Install suggested plugins
- Create the first admin user
- Finish the setup

---

## Step 4 - Understand the architecture

In this setup:

- The controller stores Jenkins state such as:
    - jobs
    - build logs
    - credentials
    - plugins
    - configuration files
- The agent executes builds in its own workspace and handles runtime tasks.

This separation is important because it allows you to:

- run builds on different machines
- isolate workloads
- scale CI/CD capacity
- keep the controller lightweight

---

## Step 5 - Configure the Jenkins agent node

In Jenkins:

1. Go to Manage Jenkins
2. Open Nodes and Clouds
3. Click New Node
4. Enter the node name:

```text
jenkins-agent-1
```

Use the following settings:

- Permanent Agent
- Remote root directory:

```text
/home/jenkins
```

- Labels:

```text
docker
```

- Usage:

```text
Use this node as much as possible
```

- Launch method:

```text
Launch agents via SSH
```

- Host:

```text
jenkins-agent-1
```

- Port:

```text
22
```

- Credentials:
    - Create a new credential of type: SSH Username with private key
    - Username:

```text
jenkins
```

- Private key:
    - Paste the complete contents of the private key from the controller or repository
    - Leave passphrase empty

The agent Dockerfile already prepares the agent with:

- a Jenkins user
- SSH server
- a public key in authorized_keys

---

## Step 6 - Use the SSH key pair correctly

The agent expects the public key to be installed in the agent’s authorized keys file. The controller must use the matching private key.

The public key is stored here:

```text
ssh/jenkins_agent_key.pub
```

The private key is stored here:

```text
ssh/jenkins_agent_key
```

Important:

- Jenkins should receive the private key in the credential
- The agent should receive the matching public key
- Do not paste the public key into Jenkins credentials

---

## Step 7 - Verify SSH access manually

From the controller container, test SSH access to the agent:

```bash
docker exec -u jenkins jenkins-controller ssh \
  -i /var/jenkins_home/.ssh/jenkins_agent_1 \
  -o IdentitiesOnly=yes \
  -o StrictHostKeyChecking=yes \
  jenkins@jenkins-agent-1 'whoami && hostname && java -version'
```

Expected output should include:

```text
jenkins
jenkins-agent-1
openjdk version "21..."
```

---

## Step 8 - Verify the key fingerprints

On the controller, inspect the private key fingerprint:

```bash
docker exec -u jenkins jenkins-controller bash -c '
ssh-keygen -y -f /var/jenkins_home/.ssh/jenkins_agent_1 | ssh-keygen -lf -
'
```

On the agent, inspect the authorized keys fingerprint:

```bash
docker exec -u jenkins jenkins-agent-1 \
ssh-keygen -lf /home/jenkins/.ssh/authorized_keys
```

The fingerprints must match.

Also compare the public keys:

```bash
docker exec -u jenkins jenkins-controller cat /var/jenkins_home/.ssh/jenkins_agent_1.pub
```

```bash
docker exec -u jenkins jenkins-agent-1 cat /home/jenkins/.ssh/authorized_keys
```

They should be identical.

---

## Step 9 - Fix agent SSH permissions if needed

If the agent refuses the connection, verify the permissions:

```bash
docker exec -u root jenkins-agent-1 bash -c '
stat -c "%U:%G %a %n" \
  /home/jenkins \
  /home/jenkins/.ssh \
  /home/jenkins/.ssh/authorized_keys
'
```

Expected values:

```text
jenkins:jenkins 755 /home/jenkins
jenkins:jenkins 700 /home/jenkins/.ssh
jenkins:jenkins 600 /home/jenkins/.ssh/authorized_keys
```

If needed, correct them with:

```bash
docker exec -u root jenkins-agent-1 bash -c '
chown jenkins:jenkins /home/jenkins
chown -R jenkins:jenkins /home/jenkins/.ssh
chmod 755 /home/jenkins
chmod 700 /home/jenkins/.ssh
chmod 600 /home/jenkins/.ssh/authorized_keys
'
```

---

## Step 10 - Create or update the Jenkins credential correctly

In Jenkins:

1. Open Manage Jenkins
2. Go to Credentials
3. Open System → Global credentials
4. Edit or create the credential named:

```text
jenkins-agent-1-key
```

Use:

- Kind: SSH Username with private key
- Scope: Global
- ID: jenkins-agent-1-key
- Username: jenkins
- Private Key: Enter directly
- Paste the complete private key
- Leave passphrase empty

This is the most common point of failure. If the username or private key does not exactly match the working SSH command, Jenkins will reject the agent connection.

---

## Step 11 - Relaunch the agent

After saving the credential:

1. Open the node configuration page
2. Click Save
3. Click Relaunch agent

If everything is correct, the node should become online.

---

## Troubleshooting tips

### 1. Username mismatch

If the agent logs show messages like:

```text
Failed publickey for root
```

or:

```text
Failed publickey for jenkins
```

then the credential username or private key is not correct.

### 2. Wrong key pasted into Jenkins

Jenkins must receive the private key, not the public key.

- Correct: private key block starting with:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
```

- Incorrect: the `.pub` file content

### 3. SSH daemon logs

For richer logs, the agent container can be started with:

```bash
docker compose build jenkins-agent-1
docker compose up -d --force-recreate jenkins-agent-1
```

Then view logs:

```bash
docker logs -f jenkins-agent-1
```

---

## Useful Docker commands

```bash
docker compose ps
docker compose logs -f jenkins-controller
docker compose logs -f jenkins-agent-1
docker compose down
docker compose down -v
```

---

## Summary

This repository gives you a complete local lab for understanding Jenkins master-slave architecture. You will learn how to:

- run Jenkins in Docker
- attach a remote agent
- configure SSH-based agent authentication
- verify connectivity
- troubleshoot authentication issues confidently

Once the agent is connected, you can begin creating freestyle jobs, parameterized builds, and pipeline-based automation on top of this foundation.