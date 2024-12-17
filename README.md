### Use Systems Manager Session Manager port forwarding to connect to EC2 instance through RDP


Port forwarding is a feature of Systems Manager Session Manager. This feature allows you to create tunnels between your local system and instances that are deployed in **private subnets**. You don't need to open inbound ports or configure bastion hosts. You can use this feature to connect to your Amazon EC2 Windows instances through RDP when keeping inbound access blocked on security groups.


#### Prerequisites
1. Install [AWS CLI version 1.16.12](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) or later on your local computer
2. [Configure the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html) with your aws account
3. Install the [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) that's specific to your operating system on your local computer.


### Launch EC2 Windows instance

1. IAM Role for SSM: Ensure you have an IAM role with the SSM Managed Instance Core policy.
    - Go to the IAM Console.
    - Create an IAM Role for EC2 with the following policy:
    ```
        AmazonSSMManagedInstanceCore
    ```
    - Attach this IAM role to the EC2 instance during launch.

2. Launch a Windows EC2 instance, make sure
    - You **NEED NOT** specify a key pair. However you can always provide this, if you need to retrieve password later.
    - You **DO NOT** assign a Public IP
    - You **DO NOT** allow RDP port 3389 or any other port to this security group
    - You **MUST** assign the previously create iam role to the instance in Advanced Details -> IAM instance profile 
    - You **CAN** specify an administrator password to use on the instance
```
<powershell>
# Set the Administrator password
$Password = "MySecurePassword123!"  # Replace with your desired password
net user Administrator $Password
</powershell>
```

3. Ensure AWS Systems Manager Agent (SSM Agent) version 2.3.672.0 or later is installed and running.
    - The SSM Agent is pre-installed on most Windows AMIs provided by AWS.
    - If using a custom AMI, ensure the SSM Agent is installed and running.

4. Launch the instance and note the instance_id


### Establish the connection
1. Establish the port forwarding session from your local computer to the EC2 instance:

    Run the following command on your local computer:
    ```
    aws ssm start-session --target <instanceid> --document-name AWS-StartPortForwardingSession --parameters "localPortNumber=55678,portNumber=3389"
    ```
    This establishes a tunnel from port 55678 in your local computer to port 3389 (RDP port) in the remote EC2 instance.

    This command also assumes that the EC2 instance operating system is configured to accept RDP connections on the default port 3389. Replace the values for localPortNumber and portNumber with your values.

    If your session connection is successful, then you get the following message:
    ```
    Starting session with SessionId: xxxxx-01234567891011abc
    Port 55678 opened for sessionId xxxxx-01234567891011abc
    Waiting for connections...
    ```

2. Use the tunnel to connect to the remote EC2 instance through RDP:

    Using a local RDP client, connect to localhost:55678. This forwards traffic to the remote port 3389 on the EC2 instance. 
    
3. Make sure to use the correct username (default: Administrator) and the password your specified earlier

    After connecting to the RDP session, AWS CLI indicates that the connection is established over the tunnel with the following message:
    ```
    Connection accepted for session [xxxxx-01234567891011abc]
    ```


# Troubleshoot the connection
If your session fails to connect, then it might be because of the following reasons:

1. **Insufficient permissions (AccessDeniedException):** Verify that the user that's starting the session has the necessary permissions for Session Manager.
2. **Instance not connected (TargetNotConnected):** 
    - Check if the instance is available. It could be as simple as misconfigured region in your cli command
    ```
    aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId, State.Name, PublicIpAddress, PrivateIpAddress, Tags[?Key=='Name'].Value|[0]]" --output table
    ``` 
    - The specified target node isn't configured for Session Manager. Verify that the target node is fully configured for Session Manager and is reporting as Online in the Systems Manager Fleet Manager console. For more information, see Managed node not available or not configured for Session Manager.
3. **Session Manager plugin not found:** Make sure that the Session Manager plugin is installed on your local machine. For more information, see Install the Session Manager plugin for the AWS CLI.
4. **Session connects successfully, but RDP client can’t connect:** Check if the default RDP port was changed in the target instance. Replace the portNumber parameter value with your value. For more information, see Check the RDP listener port on the Microsoft website.
5. **Retrieve Default Password (If Not Using User Data)**
If you didn’t set the password in User Data, retrieve the default password:
- Go to the EC2 Console.
- Select the instance → Actions → Monitor and troubleshoot → Get Windows Password.
- Provide the private key for the key pair used during launch.
- The decrypted Administrator password will be shown.


# SSH to EC2 Linux instance
While Linux instance obviously will not have the remote desktop options available, you can still use port forwarding to connect to the instance without ssh key pair using
```
aws ssm start-session --target <instance-id> --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":["22"],"localPortNumber":["2222"]}'
```