# Nginx Reverse Proxy & Secure Backend Deployment on AWS

This repository contains the setup and configuration files for a realistic DevOps architecture demonstrating how to securely deploy a backend Node.js application behind an Nginx Reverse Proxy in AWS.

In this project, the backend application is completely hidden from the public internet and can only be accessed through the reverse proxy, which acts as the single secure entry point.

## 🏗️ Architecture Overview

The infrastructure is built inside a custom AWS Virtual Private Cloud (VPC) with the following components:

* **Public Subnet:** Hosts the Nginx Reverse Proxy EC2 instance (accessible via internet).
* **Private Subnet:** Hosts the Node.js Backend EC2 instance (no public IP, hidden from internet).
* **NAT Gateway:** Allows the private backend server to download necessary packages (like Node.js and NPM) without exposing it to incoming internet traffic.
* **Security Groups:** Configured so the backend server *only* accepts SSH and HTTP traffic from the Proxy Server's Security Group.

## ⚙️ Prerequisites

* An active AWS Account.
* Basic knowledge of AWS EC2, VPC, and Linux commands.
* An SSH key pair downloaded to your local machine.

## 🚀 Step-by-Step Implementation

### Step 1: Network Setup (AWS VPC)

1. Go to the AWS VPC Dashboard and click **Create VPC**.
2. Select **VPC and more**.
3. Create 1 Public Subnet and 1 Private Subnet across 1 Availability Zone.
4. Add 1 NAT Gateway (in the selected AZ).
5. Click **Create**.

### Step 2: Configure Security Groups

1. **Proxy-SG:**
* Inbound Rules: Allow `SSH (22)` and `HTTP (80)` from `Anywhere-IPv4`.


2. **Backend-SG:**
* Inbound Rules: Allow `SSH (22)` and `Custom TCP (8080)` but set the **Source** to the `Proxy-SG` ID instead of an IP address.



### Step 3: Launch EC2 Instances

1. **Proxy Server:** Launch an Ubuntu 24.04 instance in the **Public Subnet** and attach `Proxy-SG`. Enable auto-assign Public IP.
2. **Backend Server:** Launch an Ubuntu 24.04 instance in the **Private Subnet** and attach `Backend-SG`. Do **not** assign a Public IP.

### Step 4: Backend Setup (Node.js)

Since the backend server has no public IP, connect to it by first SSH-ing into the Proxy Server, and then using the private IP to SSH into the Backend Server.

1. Install Node.js (v21+):
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 21

```


2. Clone your application code and install dependencies:
```bash
git clone <your-repo-url>
cd <your-repo-folder>
npm install

```


3. Start the application (running on port 8080):
```bash
node server.js

```



### Step 5: Proxy Setup (Nginx)

Connect to your Proxy Server via its Public IP.

1. Install Nginx:
```bash
sudo apt update
sudo apt install nginx -y

```


2. Remove the default configuration:
```bash
sudo rm /etc/nginx/sites-enabled/default

```


3. Create a new configuration file:
```bash
sudo nano /etc/nginx/sites-enabled/proxy

```


4. Add the following reverse proxy configuration (replace `10.0.x.x` with your Backend Server's Private IP):
```nginx
server {
    listen 80;
    server_name "";

    location / {
        proxy_pass http://10.0.x.x:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```


5. Reload Nginx to apply changes:
```bash
sudo systemctl reload nginx

```



## 🧪 Testing the Deployment

Open a web browser and navigate to the **Public IP** of your Nginx Proxy Server:

```
http://<Proxy-Public-IP>

```

You should successfully see the Node.js application rendered, confirming that traffic is being routed securely from the internet -> proxy -> private backend.

## ⚠️ Important Note on AWS Costs (Cleanup)

To avoid unnecessary AWS charges, make sure to delete your resources after testing:

1. Terminate both EC2 Instances.
2. Delete the **NAT Gateway** (This charges an hourly rate even if idle).
3. Release any associated Elastic IPs.
4. Delete the VPC.
