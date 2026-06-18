# Troubleshooting Guide - Hybrid Cloud 3-Tier Architecture

This guide covers common issues and solutions encountered during the deployment and verification of the AWS/On-Prem hybrid lab.

## 1. AWS CloudFormation Deployment
### Circular Dependency Error
*   **Symptom**: `An error occurred (ValidationError) when calling the CreateStack operation: Circular dependency between resources...`
*   **Cause**: This usually happens when a Security Group (SG) has an ingress rule referencing itself or another resource that in turn references the SG.
*   **Fix**: Move the self-referencing rule into a standalone `AWS::EC2::SecurityGroupIngress` resource rather than defining it inside the `AWS::EC2::SecurityGroup` properties.

## 2. AWS Systems Manager (SSM) Access
### CLI Session Plugin Missing
*   **Symptom**: `SessionManagerPlugin is not found.`
*   **Cause**: The local machine is missing the required plugin for the AWS CLI to open SSM sessions.
*   **Fix**: 
    1.  Install the [Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).
    2.  **OR** use the **AWS Management Console**: Navigate to EC2 > Instances > Select Instance > Connect > Session Manager.

## 3. Site-to-Site VPN Connectivity
### VPN is "UP" but Ping Fails (Asymmetric Routing)
*   **Symptom**: `sudo ipsec status` shows `ESTABLISHED`, but `ping 10.x.x.x` results in 100% packet loss.
*   **Cause**: Traffic is exiting via one VPN tunnel and returning via the other. Linux drops these packets by default due to Reverse Path Filtering (`rp_filter`).
*   **Fix**: Disable `rp_filter` on the on-premises gateway instance:
    ```bash
    sudo sysctl -w net.ipv4.conf.all.rp_filter=0
    sudo sysctl -w net.ipv4.conf.default.rp_filter=0
    sudo sysctl -w net.ipv4.conf.ens5.rp_filter=0 # Replace ens5 with your interface
    ```

### VPN is "UP" but Ping Still Fails (Missing Route Propagation)
*   **Symptom**: `sudo ipsec status` shows `ESTABLISHED`, `rp_filter` is disabled, but ping from on-prem to AWS private IPs still gets 100% packet loss in both directions.
*   **Cause**: The AWS App and/or DB route tables are missing the propagated route to `192.168.1.0/24` (on-prem CIDR). This happens because `AWS::EC2::VPNGatewayRoutePropagation` silently ignores multiple `RouteTableIds` entries — only the first one gets propagation applied.
*   **Diagnosis**: Check if `192.168.1.0/24` appears in the App route table:
    ```bash
    aws ec2 describe-route-tables --route-table-ids <app-rt-id> \
      --query "RouteTables[*].Routes[?DestinationCidrBlock=='192.168.1.0/24']" --output table
    ```
*   **Fix**: Enable propagation manually for each missing route table:
    ```bash
    aws ec2 enable-vgw-route-propagation --route-table-id <app-rt-id> --gateway-id <vgw-id>
    aws ec2 enable-vgw-route-propagation --route-table-id <db-rt-id> --gateway-id <vgw-id>
    ```
*   **Permanent Fix**: In CloudFormation, use a **separate** `AWS::EC2::VPNGatewayRoutePropagation` resource for each route table — do not list multiple `RouteTableIds` in a single resource.

### Missing IP Forwarding
*   **Symptom**: Traffic cannot traverse the VPN instance to reach internal subnets.
*   **Fix**: Enable IPv4 forwarding:
    ```bash
    sudo sysctl -w net.ipv4.ip_forward=1
    ```

## 4. Database Access (PostgreSQL)
### Bash Shell Errors in Password
*   **Symptom**: `-bash: !5: event not found` or `syntax error near unexpected token ')'`.
*   **Cause**: The database password contains special characters (`!`, `(`, `)`, `?`) that Bash interprets as shell operators.
*   **Fix**: Always wrap the password in **single quotes** (`'`) when setting environment variables:
    ```bash
    export PGPASSWORD='your_password_here'
    ```

### Missing Client Tools
*   **Symptom**: `psql: command not found` or `You must install at least one postgresql-client-<version> package`.
*   **Fix (Ubuntu)**:
    ```bash
    sudo apt-get update && sudo apt-get install -y postgresql-client
    ```
*   **Fix (Amazon Linux)**:
    ```bash
    sudo dnf install postgresql16 -y
    ```
