# How to Attach an AWS EC2 Instance as a Jenkins Slave

This guide will walk you through attaching an AWS EC2 instance (assuming a Linux OS like Amazon Linux 2 or Ubuntu) as a slave node to your Jenkins master running on your local Windows laptop.

## Prerequisites

1.  **Running Jenkins Master:** Your Jenkins master is up and running on your Windows laptop.
2.  **AWS EC2 Instance:** You have a running EC2 instance in AWS.
3.  **Java Installed on EC2:** Jenkins requires Java to run the agent on the slave.
    *   Connect to your EC2 instance via SSH and install Java (e.g., `sudo yum install java-17-amazon-corretto` for Amazon Linux or `sudo apt install openjdk-17-jre` for Ubuntu).
4.  **Security Group Configuration:**
    *   Find your laptop's public IP address (you can google "What is my IP").
    *   Go to your AWS EC2 console, edit the Security Group attached to your EC2 instance.
    *   Add an Inbound Rule to allow **SSH (Port 22)** from your laptop's public IP address. *(Alternatively, you can allow `0.0.0.0/0` temporarily for testing, but it is much less secure)*.
5.  **SSH Key Pair:** You need the private key (`.pem` file) that you used when launching the EC2 instance.

## Step 1: Install SSH Build Agents Plugin in Jenkins

1.  Open your Jenkins dashboard (e.g., `http://localhost:8080`).
2.  Go to **Manage Jenkins** > **Plugins** (or Manage Plugins).
3.  Click on the **Available plugins** tab.
4.  Search for **SSH Build Agents**.
5.  If it's not installed, check the box and click **Install without restart** (or "Download now and install after restart").

## Step 2: Add EC2 Credentials in Jenkins

1.  From the Jenkins dashboard, go to **Manage Jenkins** > **Credentials**.
2.  Click on **System** under the Store column, then click **Global credentials (unrestricted)**.
3.  Click **Add Credentials** on the top left.
4.  Fill in the details:
    *   **Kind:** SSH Username with private key
    *   **Scope:** Global
    *   **ID:** `ec2-ssh-key` (or any name you prefer)
    *   **Description:** Key for EC2 Slave
    *   **Username:** The default username for your EC2 instance (`ec2-user` for Amazon Linux, `ubuntu` for Ubuntu).
    *   **Private Key:** Select **Enter directly**. Click **Add**, open your `.pem` file in a text editor (like Notepad), copy the entire contents (including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`), and paste it into the box.
5.  Click **Create**.

## Step 3: Add the EC2 Node as a Slave

1.  Go to **Manage Jenkins** > **Nodes** (or Manage Nodes and Clouds).
2.  Click **New Node** on the left menu.
3.  **Node Name:** `AWS-EC2-Slave` (or any recognizable name).
4.  Select **Permanent Agent** and click **Create**.
5.  Configure the Node details:
    *   **Description:** AWS EC2 Slave Node
    *   **Number of executors:** `1` or `2` (depending on EC2 size).
    *   **Remote root directory:** `/home/ec2-user/jenkins` (or `/home/ubuntu/jenkins`). *This directory will be created on the EC2 instance if it doesn't exist.*
    *   **Labels:** `linux` `ec2` (Use labels to restrict jobs to this node later).
    *   **Usage:** Use this node as much as possible.
    *   **Launch method:** Select **Launch agents via SSH**.
        *   *If you do not see this option, double-check that the "SSH Build Agents" plugin is installed and enabled.*
    *   **Host:** The **Public IPv4 address** or Public IPv4 DNS of your EC2 instance.
    *   **Credentials:** Select the `ec2-ssh-key` credential you created in Step 2.
    *   **Host Key Verification Strategy:** Select **Non verifying Verification Strategy** (this is the easiest for initial setup, though "Known hosts file" is more secure for production).
6.  Click **Save**.

## Step 4: Verify the Connection

1.  After saving, you will be taken back to the Nodes list.
2.  Click on your new node `AWS-EC2-Slave`.
3.  Click the **Launch agent** button if it hasn't started automatically.
4.  Check the **Log** on the left menu. You should see Jenkins connecting to the EC2 instance via SSH, installing the agent (`agent.jar`), and finally showing `Agent successfully connected and online`.
5.  The node icon in the Nodes list should now be solid without a red X, indicating it's online and ready to execute jobs.

## Troubleshooting

*   **Connection Timeout:** If Jenkins cannot reach the host, double-check your AWS Security Group to ensure port 22 is open to your laptop's public IP. Also, check if your Windows Firewall or router is blocking outgoing connections to port 22 (less common).
*   **Authentication Failed:** Double-check the username (`ec2-user`, `ubuntu`, etc.) and that the private key is pasted correctly without any missing characters.
*   **Java Not Found:** Ensure Java is installed on the EC2 instance and is available in the system PATH. You can specify the exact Java path in the "Advanced" settings of the Launch method if needed.

## Step 5: Test the Slave Node

Now that the slave is online, you can test it by running a simple Jenkins job on it.

### Option A: Testing with a Freestyle Project

1.  From the Jenkins dashboard, click **New Item**.
2.  Enter a name like `Test-EC2-Slave`, select **Freestyle project**, and click **OK**.
3.  In the configuration page, scroll down to the **General** tab.
4.  Check the box for **Restrict where this project can be run**.
5.  In the **Label Expression** field, type the label you gave your node (e.g., `linux` or `ec2` or exactly `AWS-EC2-Slave`). You should see a message saying "Label is serviced by 1 node".
6.  Scroll down to the **Build Steps** section.
7.  Click **Add build step** and select **Execute shell** (since the slave is a Linux EC2 instance).
8.  In the Command box, enter simple Linux commands to verify:
    ```bash
    echo "Hello from EC2 Slave!"
    hostname
    uname -a
    pwd
    ```
9.  Click **Save**.
10. Click **Build Now** on the left menu.
11. Once the build finishes, click the green checkmark next to the build number under "Build History", then click **Console Output**. You should see the output of your commands executed on the EC2 instance!

### Option B: Testing with a Pipeline

If you prefer using Jenkins Pipelines, here is a simple script to test:

1.  Create a **New Item** -> **Pipeline** -> named `Test-Pipeline`.
2.  Scroll down to the **Pipeline** section and enter the following script:
    ```groovy
    pipeline {
        agent {
            label 'linux' // Replace 'linux' with the label you assigned to the node
        }
        stages {
            stage('Test EC2 Connection') {
                steps {
                    sh 'echo "Hello from Pipeline on EC2!"'
                    sh 'hostname'
                    sh 'uptime'
                }
            }
        }
    }
    ```
3.  Click **Save** and **Build Now**. Check the logs of the "Test EC2 Connection" stage to verify the output.
