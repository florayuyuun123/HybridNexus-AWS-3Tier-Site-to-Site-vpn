# Building a Real-World AWS Networking Lab: VPC, Private Subnets, and Site-to-Site VPN
## Implementing SSM Session Manager and AWS Secrets Manager for an Enterprise-Grade Security Profile

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
aws cloudformation create-stack --stack-name lab-onprem --template-body file://onprem-simulation.yaml --parameters ParameterKey=KeyName,ParameterValue=<your-key-pair> --capabilities CAPABILITY_NAMED_IAM
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

This stack deploys the main VPC (`10.10.0.0/16`), Subnets, ALB, App instances, and a Postgres RDS database. The app instances will pull your website from a private GitHub repo at boot time.

**Prerequisite — Store your GitHub PAT in Secrets Manager:**
The EC2 instances retrieve this secret automatically on first boot. You must create it *before* launching the stack.

1. Go to [GitHub → Settings → Developer Settings → Personal access tokens → Tokens (classic)](https://github.com/settings/tokens/new)
2. Create a token with only the ✅ `repo` scope
3. Run this command (replace the token value):

```powershell
aws secretsmanager create-secret `
  --name "lab-github-token" `
  --secret-string '{"token":"ghp_YOUR_PAT_HERE"}'
```

> [!IMPORTANT]
> This secret must exist **before** you run the `create-stack` command below. The instances read it at first boot to clone the website repo.

**Command (PowerShell):**
*Replace `34.200.235.113` with the IP you copied above and `your-key-pair` with your key name.*
```powershell
aws cloudformation create-stack --stack-name lab-network --template-body file://cloudnetwork.yaml --parameters ParameterKey=EnvironmentName,ParameterValue=lab ParameterKey=KeyName,ParameterValue=<your-key-pair> ParameterKey=OnPremPublicIP,ParameterValue=34.200.235.113 ParameterKey=OnPremCIDR,ParameterValue=192.168.1.0/24 --capabilities CAPABILITY_NAMED_IAM
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

3.  **Verify Load Balancing (Fresh Connections):**
    If you refresh your browser and the IP doesn't change, it's likely due to browser "Keep-Alive." To see the ALB switch between AZs, run this from your local PowerShell:
    ```powershell
    # Run 10 requests to see the distribution
    1..10 | ForEach-Object { curl -s http://<LoadBalancerDNS> | Select-String "Hello" }
    ```

2.  **Test Database Access (SSM Session Manager):**
    We use AWS Systems Manager to securely connect to the private App instances without needing open ports or a Bastion host.

    **Get the Connection Details:**
    ```powershell
    # Get the App Instance ID
    aws cloudformation describe-stacks --stack-name lab-network --query "Stacks[0].Outputs[?OutputKey=='AppInstance1Id'].OutputValue" --output text --no-cli-pager

    # Get the RDS Endpoint
    aws cloudformation describe-stacks --stack-name lab-network --query "Stacks[0].Outputs[?OutputKey=='DBEndpoint'].OutputValue" --output text --no-cli-pager
    ```

    **Start a Session & Test RDS:**

    ```bash
    # 1. Connect to the instance
    aws ssm start-session --target <AppInstance1Id>
    ```

    > [!NOTE]
    > If `aws ssm start-session` fails locally due to a missing Session Manager plugin, connect directly through the **AWS Management Console**: EC2 > Instances > Select Instance > Connect > Session Manager.

    ```bash
    # 2. Inside the session, install psql if not present (Amazon Linux 2023)
    sudo dnf install -y postgresql15

    # 3. Get the DB password from Secrets Manager
    # (The instance has an IAM role that allows this)
    DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id lab-db-secret --query SecretString --output text | jq -r .password)
    echo $DB_PASSWORD  # Verify the password was retrieved

    # 4. Connect to the database using the retrieved password
    export PGPASSWORD=$DB_PASSWORD
    psql -h <DBEndpoint> -U labadmin -d postgres
    ```

    **Success Criteria:**
    - You see your website with a 🟢 **"Served by: `10.10.x.x` | AZ: `AZ-A`"** badge in the bottom-right corner.
    - Refreshing the browser switches the badge to **AZ-B** (different IP), confirming the ALB is load balancing across both Availability Zones.
    - You successfully log into the database via SSM and see the `postgres=>` prompt.

## 4. Configure the VPN Connection (Manual Steps)
*Goal: Configure the "On-Prem" router to "dial" the AWS VPN.*

AWS is ready and waiting. Now we must tell the strongSwan software how to connect.

1.  **Get the VPN Configuration:**
    *   Navigate to **AWS Console** -> **VPC** -> **Site-to-Site VPN Connections**.
    *   Select `lab-vpn` and click **Download Configuration**.
    *   Select **Generic** for Vendor, **Generic** for Platform, and **strongSwan** for Software.

2.  **SSH into your "On-Prem" EC2:**
    ```bash
    ssh -i <your-key-pair>.pem ubuntu@<ON_PREM_PUBLIC_IP>
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
        auto=add
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

    **Step 4c: Enable Automatic Failover (Monitor Script)**
    Because this is a Policy-Based VPN, having both tunnels active simultaneously with the same subnets can cause routing conflicts in Linux. To achieve automatic failover, we can run a cron job that monitors Tunnel 1 and automatically brings up Tunnel 2 if Tunnel 1 goes down.
    
    ```bash
    sudo bash -c 'cat > /usr/local/bin/vpn-failover.sh <<"EOF"
    #!/bin/bash
    # Check if Tunnel 1 is currently established
    if ! sudo ipsec status Tunnel1 | grep -q "ESTABLISHED"; then
        # Tunnel 1 is down, bring up Tunnel 2
        sudo ipsec up Tunnel2
    else
        # Tunnel 1 is up. If Tunnel 2 is also up, bring it down to prevent conflicts
        if sudo ipsec status Tunnel2 | grep -q "ESTABLISHED"; then
             sudo ipsec down Tunnel2
        fi
    fi
    EOF'
    sudo chmod +x /usr/local/bin/vpn-failover.sh

    # Add to root's crontab to run every minute
    sudo bash -c '(crontab -l 2>/dev/null; echo "* * * * * /usr/local/bin/vpn-failover.sh") | crontab -'
    ```

4.  **Restart & Verify:**
    ```bash
    sudo ipsec restart
    sudo ipsec status
    ```

    **Step 4d: Test the Failover**
    To ensure the automatic failover script works correctly, simulate a failure of Tunnel 1:
    ```bash
    # 1. Bring down Tunnel 1 manually
    sudo ipsec down Tunnel1

    # 2. Run the failover script (or wait 1 minute for cron)
    sudo /usr/local/bin/vpn-failover.sh

    # 3. Verify Tunnel 2 is now ESTABLISHED
    sudo ipsec status
    ```

## 5. Final Verification
*Goal: Prove that traffic can pass privately and securely across the hybrid boundary.*

1.  **Get App Instance Private IP (run from your local machine):**
    ```powershell
    aws cloudformation describe-stacks --stack-name lab-network --query "Stacks[0].Outputs[?OutputKey=='AppInstance1Id'].OutputValue" --output text --no-cli-pager
    ```
    Then get the private IP:
    ```powershell
    aws ec2 describe-instances --instance-ids <AppInstance1Id> --query "Reservations[*].Instances[*].PrivateIpAddress" --output text
    ```

2.  **Get DB Private IP (run from on-prem instance):**
    RDS does not expose a static private IP — resolve it from the DNS endpoint:
    ```bash
    dig +short <DBEndpoint>
    ```
    Use the returned IP (e.g., `10.10.20.5`) for ping testing only. Always use the DNS endpoint for actual connections.

3.  **Ping from On-Prem to AWS:**
    Wait **30-90 seconds** after strongSwan is configured before testing — the VPN tunnel and route propagation need time to settle.
    ```bash
    ping <AppInstancePrivateIP>
    ping <DB_Private_IP>
    ```

4.  **Access Database from On-Prem:**
    Test the direct database connection over the VPN tunnel:
    ```bash
    # Install psql client if needed
    sudo apt update && sudo apt install -y postgresql-client

    # Connect using the DNS endpoint (not the IP)
    DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id lab-db-secret --query SecretString --output text | jq -r .password)
    export PGPASSWORD=$DB_PASSWORD
    psql -h <DBEndpoint> -U labadmin -d postgres
    ```
**Success:** A reply and a successful login confirm the "Company Problem" is solved: Secure, private hybrid connectivity is fully operational.

### Useful Database Commands
Once connected to the RDS Postgres instance (`postgres=>`), use these commands to explore:

| Command | Action |
| :--- | :--- |
| `\l` | List all databases. |
| `\dt` | List all tables in the current database. |
| `\conninfo` | Show current connection details (IP/SSL). |
| `SELECT version();` | Check Postgres server version. |
| `\q` | Quit the database. |
| `\dt` | List all tables in the current database. |

## 6. Best Practices & Recommendations
Based on the implementation of this lab, here are several "Production-Ready" recommendations:

1.  **Embrace Zero-SSH:** Moving from Bastion/SSH to **SSM Session Manager** significantly reduces your attack surface.
2.  **Cost Optimization:**
    *   **NAT Gateway:** While required for initial setup, it is the most expensive "idle" resource. Consider using a **Golden AMI** to eliminate the need for an outbound NAT Gateway entirely.
    *   **Endpoint Audit:** Always remove Interface Endpoints (like Secrets Manager) if your application logic doesn't actively use them to save ~$7/mo.
3.  **Dynamic Secrets:** Never hardcode passwords. The integration with **AWS Secrets Manager** in this project ensures that credentials are randomized and retrieved securely at runtime.
4.  **Logging & Visibility:** Ensure **VPC Flow Logs** are active to audit all traffic crossing your hybrid boundary (Site-to-Site VPN).

## 7. Cleanup
*Goal: Remove all resources to stop billing.*

```powershell
# Delete the CloudFormation stacks (this removes EC2, RDS, VPN, ALB, etc.)
aws cloudformation delete-stack --stack-name lab-network
aws cloudformation delete-stack --stack-name lab-onprem

# Delete the GitHub PAT secret from Secrets Manager
aws secretsmanager delete-secret --secret-id lab-github-token --force-delete-without-recovery
```

> [!NOTE]
> Stack deletion order matters — `lab-network` and `lab-onprem` can be deleted in parallel. The `lab-db-secret` (RDS password) is deleted automatically with the `lab-network` stack. The `lab-github-token` secret is standalone and must be deleted separately with the command above.
