# Secure Hybrid 3-Tier Web Application Hub

This guide documents the successful deployment of a production-ready 3-tier architecture with a Site-to-Site VPN Simulation.

## Company Integration Problem
**Scenario:**  
A mid-sized enterprise, "TechCorp," has a significant on-premises data center hosting legacy applications and sensitive customer databases. They are migrating their frontend and new microservices to AWS to leverage scalability and managed services.

**The Problem:**  
TechCorp needs a secure, private, and reliable connection between their AWS VPC and their on-premises data center.  
- Public internet exposure for internal database traffic is a security violation.  
- Dedicated lines (Direct Connect) are too expensive and slow to provision for the initial pilot phase.  
- Manual configuration of VPNs is error-prone and has led to outages in previous attempts.

## Reason for Project
This project implements a **Site-to-Site VPN** solution using AWS-managed VPN services on the cloud side and a software-based Customer Gateway (strongSwan) on the simulated on-premise side.

**Project Goals:**  
1.  **Security:** Encrypt all traffic between Cloud and On-Prem using IPsec.  
2.  **Automation:** Use Infrastructure as Code (CloudFormation) to deploy the network, reducing human error.  
3.  **Simulation:** Create a realistic testbench to validate the VPN configuration before deploying to production hardware.  
4.  **Reliability:** Enable Route Propagation to automatically update routing tables.

---

## 1. Deploy the "On-Prem" Simulator
*Goal: Create a simulated "Data Center" environment to test the VPN connection without needing physical hardware.*

This stack deploys a separate VPC (`192.168.0.0/16`) and an Ubuntu EC2 instance acting as the Customer Gateway.

**Command (PowerShell):**
```powershell
aws cloudformation create-stack --stack-name lab-onprem --template-body file://onprem-simulation.yaml --parameters ParameterKey=KeyName,ParameterValue=argo-key-pair --capabilities CAPABILITY_NAMED_IAM
```

**Why this step?**  
Real-world testing requires two distinct networks. This simulation mimics the customer's on-site network router.

**Wait for completion & Get Public IP:**
Run this command to retrieve the Public IP you will need for the next step:
```powershell
aws cloudformation describe-stacks --stack-name lab-onprem --query "Stacks[0].Outputs[?OutputKey=='OnPremPublicIP'].OutputValue" --output text --no-cli-pager
```
*Copy this IP address.* (Example: `34.200.235.113`)

## 2. Deploy the Lab Network (VPN Side)
*Goal: Provision the AWS environment with a full 3-tier application stack and VPN gateway.*

This stack deploys the main VPC (`10.10.0.0/16`), Subnets, ALB, Bastion, App instances, and a Postgres RDS database.

**Command (PowerShell):**
*Replace `34.200.235.113` with the IP you copied above and `argo-key-pair` with your key name.*
```powershell
aws cloudformation create-stack --stack-name lab-network --template-body file://cloudnetwork.yaml --parameters ParameterKey=EnvironmentName,ParameterValue=lab ParameterKey=KeyName,ParameterValue=argo-key-pair ParameterKey=OnPremPublicIP,ParameterValue=34.200.222.242 ParameterKey=OnPremCIDR,ParameterValue=192.168.1.0/24 ParameterKey=UserIP,ParameterValue=0.0.0.0/0 --capabilities CAPABILITY_NAMED_IAM
```
*(Note: If the stack already exists, use `update-stack` instead.)*

**Check Deployment Progress:**
Since this stack includes an RDS database, it will take **12-15 minutes** to complete. Monitor the status with this command:
```powershell
aws cloudformation describe-stacks --stack-name lab-network --query "Stacks[0].StackStatus" --output text
```
Wait until it returns **`CREATE_COMPLETE`** before proceeding.

## 3. Verify the 3-Tier Application (Web & DB)
*Goal: Ensure the cloud side is working before connecting the VPN.*

1.  **Access the Website (ALB):**
    Get the ALB DNS from the stack outputs and paste it into your browser.
    ```powershell
    aws cloudformation describe-stacks --stack-name lab-network --query "Stacks[0].Outputs[?OutputKey=='LoadBalancerDNS'].OutputValue" --output text --no-cli-pager
    ```
    *Expected:* You should see a "Hello World" page from one of the app servers.

    > [!TIP]
    > **Connecting Error?** If you get "Connection Refused," your browser might be trying to force HTTPS. Ensure you are using `http://` (not `https://`) or try an Incognito window.

2.  **Test Database Access (Bastion):**
    First, SSH into the Bastion (using **Agent Forwarding** to pass your key), then hop to an App server to test the RDS connection.
    ```bash
    # From your local machine (ensure your key is added to ssh-agent)
    ssh -A ec2-user@<BastionPublicIP>
    
    # From Bastion
    ssh ec2-user@<AppInstancePrivateIP>
    
    # From App Instance (Test RDS)
    psql -h <DBEndpoint> -U labadmin -d postgres
    ```

    **Tip:** You can also test directly from the **Bastion** by running `sudo dnf install postgresql16 -y` first.

## 4. Configure the VPN Connection (Manual Steps)
*Goal: Configure the "On-Prem" router to "dial" the AWS VPN.*

AWS is ready and waiting. Now we must tell the strongSwan software how to connect.

1.  **Get the VPN Configuration:**
    *   Navigate to **AWS Console** -> **VPC** -> **Site-to-Site VPN Connections**.
    *   Select `lab-vpn` and click **Download Configuration**.
    *   Select **Generic** for Vendor, **Generic** for Platform, and **strongSwan** for Software.

2.  **SSH into your "On-Prem" EC2:**
    ```bash
    ssh -i argo-key-pair.pem ubuntu@<ON_PREM_PUBLIC_IP>
    ```

3.  **Update strongSwan Config:**
    Reference the downloaded file for `<TUNNEL_IPs>` and `<PSKs>`.
    
    **Step 4a: Configure `ipsec.conf`**
    ```bash
    sudo bash -c 'cat > /etc/ipsec.conf <<EOF
    # ipsec.conf - strongSwan IPsec configuration file
    config setup
        charondebug="ike 1, knl 1, cfg 0"
        uniqueids=no
    conn %default
        keyingtries=%forever
        dpddelay=10s
        dpdtimeout=30s
        dpdaction=restart
    conn Tunnel1
        type=tunnel
        auto=start
        keyexchange=ikev1
        authby=secret
        left=%defaultroute
        leftid=<ON_PREM_PUBLIC_IP>      # Your On-Prem EC2 Public IP
        leftsubnet=192.168.1.0/24
        right=<TUNNEL1_AWS_IP>          # AWS Tunnel 1 Outside IP
        rightsubnet=10.10.0.0/16
        ike=aes128-sha1-modp1024
        esp=aes128-sha1-modp1024
        ikelifetime=8h
        keylife=1h
    conn Tunnel2
        type=tunnel
        auto=start
        keyexchange=ikev1
        authby=secret
        left=%defaultroute
        leftid=<ON_PREM_PUBLIC_IP>      # Your On-Prem EC2 Public IP
        leftsubnet=192.168.1.0/24
        right=<TUNNEL2_AWS_IP>          # AWS Tunnel 2 Outside IP
        rightsubnet=10.10.0.0/16
        ike=aes128-sha1-modp1024
        esp=aes128-sha1-modp1024
        ikelifetime=8h
        keylife=1h
    EOF'
    ```

    **Step 4b: Configure `ipsec.secrets`**
    ```bash
    sudo bash -c 'cat > /etc/ipsec.secrets <<EOF
    <ON_PREM_PUBLIC_IP> <TUNNEL1_AWS_IP> : PSK "<TUNNEL1_PSK>"
    <ON_PREM_PUBLIC_IP> <TUNNEL2_AWS_IP> : PSK "<TUNNEL2_PSK>"
    EOF'
    ```

4.  **Restart & Verify:**
    ```bash
    sudo ipsec restart
    sudo ipsec status
    ```

## 5. Final Verification
*Goal: Prove that traffic can pass privately and securely across the hybrid boundary.*

1.  **Ping from On-Prem to AWS:**
    From your **On-Prem** EC2, ping the private IP of an App Instance or the RDS Database.
    ```bash
    ping <AppInstancePrivateIP>
    ping <DB_IP_Address>  # (e.g., 10.10.20.x)
    ```

2.  **Access Database from On-Prem:**
    Test the direct database connection over the VPN tunnel:
    ```bash
    # install client if needed: sudo apt update && sudo apt install postgresql-client -y
    psql -h <DBEndpoint> -U labadmin -d postgres
    ```
**Success:** A reply and a successful login confirm the "Company Problem" is solved: Secure, private hybrid connectivity is fully operational.

## 6. Cleanup
*Goal: Remove resources to stop billing.*

```powershell
aws cloudformation delete-stack --stack-name lab-network
aws cloudformation delete-stack --stack-name lab-onprem
```
