# 🚀 Build Agent Setup Guide

Welcome! This guide walks you through setting up Jenkins on two AWS EC2 instances to create a **controller-agent architecture** for your builds. 🖥️⚡

---

## 1️⃣ Install Jenkins on Two EC2 Instances

Follow the official Jenkins tutorial for installing on AWS:

👉 [Jenkins AWS Installation Guide](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/#creating-a-key-pair)

---

## 2️⃣ Create & Connect to EC2 Instances

```bash
# Set permissions for your private key
chmod 400 private.pem
ls -l private.pem  # Should show read-only permission

# Connect to the instance via SSH
ssh -i private.pem ec2-user@<Public-IP>
```

![EC2 Master](./assets/EC2-Master.png)
![EC2 Slave](./assets/EC2-Slave.png)

---

## 3️⃣ Configure Security Groups (SGs)

Open the following ports on both instances’ security groups:

- 🟢 **SSH (22)**
- 🌐 **HTTP (80)**
- ⚙️ **Jenkins (8080)**

![SG Master](./assets/SG-Master.png)
![SG Slave](./assets/SG-Slave.png)

---

## 4️⃣ Configure the Jenkins Agent (Node)

### 🧑‍💻 4.1 Create a User on the Agent Instance

```bash
# SSH into the Agent instance
ssh -i privatekey.pem ec2-user@<Agent-Public-IP>

# Add a user for Jenkins ---> for more security & separates Agents
sudo adduser slave     
tail -n 3 /etc/passwd   # Verify user creation
groups                  # Check current groups

# Give 'slave' root privileges
sudo usermod -aG wheel slave
groups slave            # Verify group membership

# Set a password for the new user
sudo passwd slave
```

✨ **Ensure the following in `/etc/ssh/sshd_config`:**
```bash
PermitRootLogin no
PasswordAuthentication yes
```
Then restart SSH:
```bash
sudo systemctl restart sshd
```

#### 🔗 Test SSH Connection from Controller to Agent

```bash
# Controller -- Master
# Agent -- Slave
ssh slave@<AGENT_PUBLIC_IP>
```

---

### ⚙️ 4.2 Create a Jenkins Node (Agent) in Jenkins UI

1. Go to **Manage Jenkins** ➡️ **Nodes** ➡️ **New Node**
2. Fill in:
   - **Name:** `slave`
   - **Type:** Permanent Agent
   - **Number of executors:** `1`
   - **Remote root directory:** `/home/slave`
   - **Labels:** `slave`
   - **Usage:** Only build jobs with label expressions matching this node (or use as much as possible)
   - **Launch method:** Launch agent by connecting it to the "controller" 📌
   - **Availability:** Keep this agent online as much as possible
3. **Save**

⏳ After saving, the controller will attempt to SSH into the agent and download `agent.jar`.

---

### 🟢 Bring the Agent Online

1. If the status is **Offline**:
   - Click the node name in Jenkins
   - Return to the agent’s CLI:
     - `cd /home`
     - Copy the Unix launch command from the Jenkins UI on the controller and run it on the agent

> 💡 *If the resources are insufficient, the node will remain offline.*

![Node Online](./assets/Status-Online.png)
![Log Online](./assets/Log-Online.png)

---

## 🧪 Test the Connection

1. Create a new item (**Pipeline**) in Jenkins.
2. Under **Definition**, choose **Pipeline script**.
3. Paste and save the following:

```groovy
pipeline {
    agent {
        label 'slave'
    }
    stages {
        stage('Hello') {
            steps {
                echo 'Connection success'
            }
        }
    }
}
```
---

## 📊 Output

![Run Job](./assets/Output1.png)
![Run Job](./assets/Output2.png)
![Run Job](./assets/Output3.png)

---

## 📝 Notes

### Launch Agent by Connecting to the 'Controller' 🟡🔵

**Description:**  
The agent (machine/server that runs the jobs) connects to the controller (master). The agent itself initiates a connection to the Jenkins Master and announces its availability.

**How it works:**
- Usually uses JNLP (Java Web Start) or WebSocket.
- No need to open an SSH port for the agent.
- Suitable when agents are "behind a firewall or NAT" and cannot be accessed directly.
- The controller issues a secret (token) to the agent, which is used for authentication.

**When to use this method:**
- Agents are not accessible from the controller (e.g., on a private network).
- Agents initiate the connection, enhancing security (outbound connection).

---

## 🎉 You're All Set!

Your Jenkins master-agent setup is now ready to build and deploy projects efficiently! 🚦

---