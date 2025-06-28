---
title: "ECS Security in Action: A Lab for DevOps Enthusiasts"
description: "Learn how to build a security training ground in ECS using Falco."
category: "Security"
image: "/security.png"
date: "January 19, 2025"
author: "Yogesh Patil"
---


Picture this: You’re a DevOps engineer who just finished setting up a new ECS service. Everything looks perfect until you get that dreaded alert — someone’s trying to do something nasty in your container. Now you’re rushing to figure out what happened and how to prevent it.

But what if you could be prepared? Better yet, what if you could practice detecting and responding to security threats in a controlled environment? That’s exactly what we’re building today — a security training ground right in your ECS environment!.

### Why Container Security Monitoring Matters
When we deploy containers in production, a lot can happen:

- Sometimes applications start behaving strangely
- Resources spike unexpectedly
- Unusual network connections appear
- Files get accessed that shouldn’t be touched

The challenge? By the time we notice these issues, it’s usually because something has already gone wrong. That’s where security monitoring comes in — it helps us catch these issues early, understand what’s happening, and respond quickly.

### The Security Monitoring Challenge
Most DevOps engineers face three big challenges with container security:

- Not knowing what to monitor
- Not understanding security alerts when they come
- Not having a safe place to learn and experiment

That’s exactly what we’re solving today! We’ll build a security monitoring lab where you can:
- See exactly what suspicious activities look like in your containers
- Get alerts when something fishy happens
- Learn how to spot common security issues before they become problems

### Enter Falco: Your Container’s Security Camera

Think of Falco as your container’s security camera. It watches what’s happening inside your containers and tells you when something doesn’t look right. Want to know if someone’s trying to read sensitive files? Or if a container suddenly starts acting strange? Falco’s got your back.

### What We’re Building
Our project does something unique — it combines a purposefully vulnerable application with Falco security monitoring in different security modes. Think of it as your personal security lab in the cloud!

Project Features
- A Flask app with endpoints that simulate common attack vectors
- Three security modes: Training, Production, and Chaos
- Advanced Falco rules for attack pattern detection
- Real-time security event monitoring

### Directory Structure
\`\`\`
security-training/
├── app/
│   ├── Dockerfile
│   ├── app.py
│   ├── security_utils.py
│   └── requirements.txt
├── falco/
│   ├── Dockerfile
│   ├── falco_rules.yaml
│   └── falco.yaml (We have not keep in my example we have used default one)
├── ecs/
│   └── task-definition.json
└── README.md
\`\`\`
### 1. Building the Application

## The Flask App (app/app.py)

\`\`\`
from flask import Flask, jsonify
import subprocess
import os
import random
import time
from security_utils import execute_command, simulate_crypto_mining

app = Flask(__name__)

# Get security mode from environment variable
SECURITY_MODE = os.getenv('SECURITY_MODE', 'PRODUCTION')

def is_endpoint_enabled():
    if SECURITY_MODE == 'PRODUCTION':
        return False
    elif SECURITY_MODE == 'TRAINING':
        return True
    elif SECURITY_MODE == 'CHAOS':
        return random.choice([True, False])
    return False

@app.route('/')
def home():
    return jsonify({
        "status": "running",
        "mode": SECURITY_MODE,
        "message": "Security Training Ground Active"
    })

@app.route('/exec/<command>')
def command_exec(command):
    if not is_endpoint_enabled():
        return jsonify({"error": "Endpoint disabled in current mode"}), 403
    
    result = execute_command(command)
    return jsonify({"result": result})

@app.route('/file-access/<path>')
def file_access(path):
    if not is_endpoint_enabled():
        return jsonify({"error": "Endpoint disabled in current mode"}), 403
    
    try:
        with open(path, 'r') as file:
            return jsonify({"content": file.read()})
    except Exception as e:
        return jsonify({"error": str(e)}), 400

@app.route('/network-scan/<target>')
def network_scan(target):
    if not is_endpoint_enabled():
        return jsonify({"error": "Endpoint disabled in current mode"}), 403
    
    try:
        result = subprocess.check_output(['ping', '-c', '1', target])
        return jsonify({"result": result.decode()})
    except Exception as e:
        return jsonify({"error": str(e)}), 400

@app.route('/crypto-simulate')
def crypto_mine():
    if not is_endpoint_enabled():
        return jsonify({"error": "Endpoint disabled in current mode"}), 403
    
    simulate_crypto_mining()
    return jsonify({"status": "mining simulation started"})

@app.route('/privilege-escalate')
def privilege_escalate():
    if not is_endpoint_enabled():
        return jsonify({"error": "Endpoint disabled in current mode"}), 403
    
    try:
        os.setuid(0)  # Attempt to set root privileges
        return jsonify({"status": "privilege escalation attempted"})
    except Exception as e:
        return jsonify({"error": str(e)}), 400
file-acce
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
   \`\`\`
### Security Utils (app/security_utils.py)
\`\`\`
import subprocess
import time
import threading

def execute_command(command):
    """Execute a shell command and return the output"""
    try:
        result = subprocess.check_output(command.split(), stderr=subprocess.STDOUT)
        return result.decode()
    except subprocess.CalledProcessError as e:
        return f"Error: {e.output.decode()}"

def simulate_crypto_mining():
    """Simulate cryptocurrency mining behavior"""
    def cpu_intensive_task():
        while True:
            # Simulate CPU-intensive calculations
            [x ** 2 for x in range(10000)]
            time.sleep(0.1)
    
    # Start the simulation in a background thread
    thread = threading.Thread(target=cpu_intensive_task)
    thread.daemon = True
    thread.start()
\`\`\`
### Requirements (app/requirements.txt)
\`\`\`
flask==2.0.1
Dockerfile (app/Dockerfile)
FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

ENV SECURITY_MODE=PRODUCTION

EXPOSE 5000
CMD ["python", "app.py"]
\`\`\`

### 2. Setting Up Falco
### Custom Rules (falco/falco_rules.yaml)
\`\`\`yaml
# Base security rules
- rule: Sensitive File Access
  desc: Detects access to sensitive system files
  condition: >
    open_read and fd.name startswith /etc and
    (fd.name in (/etc/shadow, /etc/passwd, /etc/ssh/ssh_config)) 
  output: "Sensitive file access detected [file=%fd.name user=%user.name container=%container.name]"
  priority: WARNING
  tags: [filesystem]

# Command execution detection
- rule: Suspicious Command Execution
  desc: Detects execution of potentially dangerous commands
  condition: >
    spawned_process and proc.name in (sh, bash, nc, curl, wget)
  output: "Suspicious command execution [cmd=%proc.cmdline user=%user.name container=%container.name]"
  priority: WARNING
  tags: [process]

# Network scanning detection
- rule: Network Scanning Activity
  desc: Detects network scanning attempts
  condition: >
    spawned_process and proc.name = ping
  output: "Network scanning detected [cmd=%proc.cmdline container=%container.name]"
  priority: WARNING
  tags: [network]

# # Crypto mining detection
- rule: Crypto Mining Activity
  desc: Detects potential cryptocurrency mining behavior
  condition: >
    spawned_process and 
    (proc.name in (minerd, cpuminer, cpuminer-multi, cpuminer-opt) or 
    proc.cmdline contains "crypto" or 
    proc.cmdline contains "mining" or
    proc.cmdline contains "monero" or
    proc.aname in (minerd, cpuminer, cpuminer-multi, cpuminer-opt))
  output: "Potential crypto mining activity detected [container=%container.name proc=%proc.name cmdline=%proc.cmdline]"
  priority: WARNING
  tags: [mining]

# Privilege escalation detection
- rule: Privilege Escalation Attempt
  desc: Detects attempts to escalate privileges
  condition: >
    evt.type = setuid and evt.arg.uid = 0
  output: "Privilege escalation attempt detected [user=%user.name container=%container.name]"
  priority: CRITICAL
  tags: [privilege]
\`\`\`

## Dockerfile (falco/Dockerfile)
\`\`\`
FROM falcosecurity/falco:latest
COPY falco_rules.yaml /etc/falco/rules.d/
\`\`\`

### 3. ECS Task Definition
### Task Definition (ecs/task-definition.json)
\`\`\`json
{
  "family": "security-training",
  "containerDefinitions": [
    {
      "name": "vulnerable-app",
      "image": "vulnerable-app:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 5000,
          "hostPort": 5000
        }
      ],
      "environment": [
        {
          "name": "SECURITY_MODE",
          "value": "TRAINING"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/security-training",
          "awslogs-region": "us-west-2",
          "awslogs-stream-prefix": "app"
        }
      }
    },
    {
      "name": "falco-sidecar",
      "image": "custom-falco:latest",
      "essential": true,
      "privileged": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/security-training",
          "awslogs-region": "us-west-2",
          "awslogs-stream-prefix": "falco"
        }
      }
    }
  ]
}
  \`\`\`
### 4. Building and Deploying
#### Build the application:
\`\`\`
# Build the vulnerable app
cd app
docker build -t vulnerable-app:latest .

# Build Falco sidecar
cd ../falco
docker build -t custom-falco:latest .
\`\`\`

### Push to ECR:
\`\`\`

# Replace with your ECR repository
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin YOUR_ACCOUNT.dkr.ecr.us-west-2.amazonaws.com

docker tag vulnerable-app:latest YOUR_ACCOUNT.dkr.ecr.us-west-2.amazonaws.com/vulnerable-app:latest
docker push YOUR_ACCOUNT.dkr.ecr.us-west-2.amazonaws.com/vulnerable-app:latest

docker tag custom-falco:latest YOUR_ACCOUNT.dkr.ecr.us-west-2.amazonaws.com/custom-falco:latest
docker push YOUR_ACCOUNT.dkr.ecr.us-west-2.amazonaws.com/custom-falco:latest
\`\`\`

#### Deploy to ECS:
\`\`\`

aws ecs register-task-definition --cli-input-json file://ecs/task-definition.json
aws ecs create-service --cluster your-cluster --service-name security-training --task-definition security-training:1 --desired-count 1
\`\`\`

### 5. Testing the Setup

\`\`\`

Training Mode Tests
# Get your service URL
export SERVICE_URL=<your-service-url>

# Test command execution
curl "$SERVICE_URL/exec/ls"

# Test file access
curl "$SERVICE_URL/file-access/etc/passwd"
# Test network scanning
curl "$SERVICE_URL/network-scan/8.8.8.8"
# Test crypto mining simulation
curl "$SERVICE_URL/crypto-simulate"
# Test privilege escalation
curl "$SERVICE_URL/privilege-escalate"
\`\`\`


### Checking Falco Alerts
\`\`\`

# View Falco logs
aws logs get-log-events --log-group-name /ecs/security-training --log-stream-name falco/security-training/<task-id>
\`\`\`

### Real-World Security Scenarios We Monitor
Let’s look at actual incidents that inspired our monitoring setup:

- The Crypto-Mining Incident
Scenario: A compromised container suddenly shows high CPU usage
Real Example: In 2018, Tesla’s Kubernetes cluster was hijacked for crypto mining
What We Monitor: Unusual CPU patterns, unexpected processes, and network connections to mining pools

- The Data Sniffer

Scenario: Container tries to read sensitive system files
Real Example: Common in container escape attempts where attackers try to access host system files
What We Monitor: Unauthorized file access attempts, especially to /etc/passwd, SSH keys, and other sensitive files

- The Network Scout

Scenario: Container performs internal network scanning
Real Example: Seen in multi-stage attacks where compromised containers try to find other vulnerable services
What We Monitor: Unusual network patterns, port scanning attempts, and unexpected internal connections

- The Privilege Hunter

Scenario: Application tries to escalate its privileges
Real Example: CVE-2019–5736 showed how containers could be used to gain host system access
What We Monitor: Attempts to modify permissions, gain root access, or escape container boundaries
- The Supply Chain Trojan

Scenario: Container runs unexpected commands or downloads unknown files
Real Example: The 2020 SolarWinds incident showed the importance of runtime monitoring
What We Monitor: Unexpected process executions, suspicious downloads, and unusual file modifications

### Security Modes Explained
- TRAINING Mode
All vulnerable endpoints enabled
Perfect for security testing and learning
Use this to test Falco rules and practice incident response

- PRODUCTION Mode

All vulnerable endpoints disabled
Use this to demonstrate secure configuration

- CHAOS Mode

Endpoints randomly enabled/disabled
Great for testing detection systems
Simulates unpredictable attack patterns

### What’s Next?
Want to level up your security game? Try these:

- Add more attack scenarios:
Container escape attempts
Data exfiltration simulations
DNS tunneling detection

-  Enhance monitoring:

Set up Prometheus metrics for security events
Create Grafana dashboards
Implement automated response with AWS Lambda

- Improve attack pattern detection:

Add machine learning for anomaly detection
Implement MITRE ATT&CK mapping
Create custom correlation rules

### Conclusion
You’ve just built your own security training ground in ECS! This setup lets you:

Practice security monitoring in a controlled environment
Test Falco rules against real attack patterns
Understand how different types of attacks look at the container level
Remember: The best way to learn security monitoring is by doing it. With this setup, you’ll see real security events happening in real-time, understand what they look like, and learn how to respond to them. Happy monitoring! 🛡️

P.S.: Always run this in an isolated environment. We don’t want our security training to become a security incident!
      