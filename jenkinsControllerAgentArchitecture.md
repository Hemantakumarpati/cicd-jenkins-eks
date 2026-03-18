# Jenkins Controller vs Agent Architecture

The **Jenkins Controller-Agent Architecture** (formerly known as Master-Slave architecture) is a distributed model designed to handle heavy CI/CD workloads, distribute builds across different environments, and keep the Jenkins system stable.

Here is a breakdown of how the two components work together:

## 1. The Jenkins Controller (The "Brain")
The Controller is the central orchestrator of your Jenkins environment. Whenever you access the Jenkins Web UI, you are interacting with the Controller.

**Key Responsibilities:**
*   **Scheduling:** It determines *when* a job should run and *which* agent should run it.
*   **State Management:** It stores all the configurations, job definitions, pipeline scripts, and historical build logs.
*   **User Interface:** It serves the web dashboard where you configure plugins, manage security, and view build results.
*   **Plugin Management:** All plugins are installed and managed on the Controller.

> [!CAUTION] 
> **Best Practice:** You should **never** run heavy builds directly on the Controller. If a build consumes 100% of the CPU or runs out of RAM, the Jenkins Web UI will crash, and all other jobs will fail. The Controller is configured with an executor count of `0` in properly designed production environments so its sole focus is orchestration.

---

## 2. The Jenkins Agent (The "Muscle")
Agents (often called Slaves or Build Nodes) are the actual machines or containers that perform the heavy lifting. They execute the jobs scheduled by the Controller.

**Key Responsibilities:**
*   **Executing Code:** Compiling code, running unit tests, building Docker images, and deploying infrastructure.
*   **Environment Isolation:** You can have different agents for different tasks. For example, a Linux agent for building Docker containers, a Windows agent for compiling .NET code, and a macOS agent (like an AWS EC2 Mac instance) for building iOS apps.

**Types of Agents:**
*   **Static Agents:** Permanent servers (like an EC2 instance or on-premise VM) dedicated to running Jenkins jobs.
*   **Dynamic/Ephemeral Agents:** Agents that are spun up on-demand to run a single job and are then destroyed afterward (e.g., Kubernetes Pods, AWS Fargate tasks, or Docker containers). This saves money and ensures a clean build environment every time.

---

## How They Communicate
The Controller and the Agent must maintain a connection to orchestrate the build and send the console output back to the UI. There are two primary ways they communicate:

1.  **SSH (Controller to Agent):** The standard method for Linux/Unix agents. The Jenkins Controller stores SSH credentials and proactively logs into the Agent machine to start the Jenkins remoting process.
2.  **Inbound / JNLP (Agent to Controller):** The Agent reaches out to the Controller. This is typically used when the Agent is behind a strict firewall or NAT (where the Controller cannot easily initiate an SSH connection), or when using dynamic Kubernetes pods.

---

## Summary Analogy
Think of the **Controller** as a restaurant manager. The manager takes the orders (webhooks), writes the menu (pipeline scripts), and assigns tasks to the staff. 

The **Agents** are the kitchen chefs. They get the instructions from the manager, grab the ingredients (source code), do the actual heavy lifting of cooking (compiling/testing), and hand the finished product back.

---

## 3. Inbound vs Outbound Agents

When connecting a Jenkins Agent to the Jenkins Controller, the terms "Inbound" and "Outbound" refer to **which side initiates the connection**.

### A. Outbound Agents (SSH Agents)
In an **Outbound** connection, the **Jenkins Controller initiates the connection** to the Agent. 
*   **How it works:** The Controller uses SSH to physically log into the Agent machine, copies over the necessary Java binaries (`remoting.jar`), and starts the JVM process to establish the build node.
*   **When to use:** This is the standard choice for Linux/Unix static agents that have a known IP address and are easily accessible over your internal network.

**Configuration Steps (Step-by-Step):**
1.  **Prepare the Agent Machine:** Ensure the instance has Java installed and an SSH server running. Create a dedicated user (e.g., `jenkins`).
2.  **Generate SSH Keys:** Create an SSH Keypair. Place the public key in the agent user's `~/.ssh/authorized_keys`.
3.  **Add Credentials to Jenkins:** On the Controller UI, navigate to *Manage Jenkins > Credentials*. Add a new "SSH Username with private key" credential and paste the matching private key.
4.  **Create the Node:** Navigate to *Manage Jenkins > Nodes > New Node*. Give it a name and select "Permanent Agent".
5.  **Configure the Node:**
    *   Set **Remote root directory** (e.g., `/home/jenkins/workspace`).
    *   Set **Launch method** to **Launch agents via SSH**.
    *   Enter the Host IP/DNS of the Agent.
    *   Select the Credentials you created in Step 3.
    *   Set "Host Key Verification Strategy" (e.g., *Non-verifying Verification Strategy* for testing, or *Known Hosts file* for production).
6.  **Save and Launch:** Click Save, and then click "Launch Agent" on the node's status page. The Controller will SSH in and bring the agent fully online.

### B. Inbound Agents (JNLP / TCP Agents)
In an **Inbound** connection, the **Jenkins Agent initiates the connection** to the Controller.
*   **How it works:** The Controller patiently listens on a specific TCP port. You must manually run a Java command on the Agent machine (passing a secure secret token) that reaches out to the Controller to establish the handshake and connection.
*   **When to use:** Use this when the Agent is behind a strict firewall/NAT that blocks incoming SSH connections (so the Controller can't reach it), when adding a Windows agent, or when using dynamic ephemeral agents like Kubernetes Pods that spawn and connect automatically.

**Configuration Steps (Step-by-Step):**
1.  **Enable Inbound TCP Port on Controller:** Navigate to *Manage Jenkins > Security*. Under the "Agents" section, set the **TCP port for inbound agents** to a fixed port (e.g., `50000`) or "Random". *Ensure this TCP port is open on your Controller's cloud firewall/security group.*
2.  **Create the Node:** Navigate to *Manage Jenkins > Nodes > New Node*. Give it a name and select "Permanent Agent".
3.  **Configure the Node:**
    *   Set **Remote root directory** (e.g., `C:\Jenkins`).
    *   Set **Launch method** to **Launch agent by connecting it to the controller**. *(If you don't see this option, ensure the TCP port was successfully enabled in Step 1).*
    *   Click Save.
4.  **Get the Secret Command:** Jenkins will now display a connect command containing a secret string on the screen. It looks similar to this:
    ```bash
    java -jar agent.jar -url http://<controller-ip>:8080/ -secret <very-long-secret-key> -name <agent-name> -workDir "C:\Jenkins"
    ```
5.  **Prepare the Agent Machine:** Ensure Java is installed on the agent. Download the `agent.jar` file from the URL provided on the agent setup screen in Jenkins.
6.  **Launch the Agent:** Run the copied command in the Agent's terminal (Command Prompt, PowerShell, or Bash). The Agent will reach out to the Controller, authenticate using the secret, and come online. 

*(Tip: On a Windows inbound agent, you usually wrap this Java command in a Windows Service using a tool like NSSM or the native Jenkins service wrapper so that it starts automatically when the Windows machine reboots).*
