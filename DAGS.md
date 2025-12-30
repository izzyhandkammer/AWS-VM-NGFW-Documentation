# Implementing IAM Policy
We created a policy to define permissions for the firewall to read EC2 instances' tags but we never implemented it.

1. Verify that you made the policy in the IAM Dashboard
2. Create the IAM role
	1. In the dashboard click Roles → Create role
	2. Trusted entity type: AWS Service
	3. Use Case: EC2
	4. Permissions: use the policy we made
	5. Name it something like `Palo-Firewall-Role` and create it
3. Attach the role to the Firewall
	1. Go to the EC2 instances page and select the firewall instance
	2. Actions → Security → Modify IAM role
	3. Select the role from the dropdown and update

# Enabling the AWS Plugin
To implement DAGs we need to turn the firewall into an "AWS reader". It's going to continuously poll the AWS environment asking questions like "who has the tag 'Role:WebServer'?" and updates the security policies in real time.

1. Go to Device → VM-Series. You should see an "AWS CloudWatch Setup" widget. 
2. Enable monitoring
3. Go to the Plugins tab and click install on the "vm_series" plugin

# Creating a Test Instance
We need a victim instance on the VPC that has a tag
1. Launch an instance from the EC2 dashboard
2. I'm choosing to launch a t3.micro Amazon Linux
3. Put it in your VPC in the Trust subnet.
4. TAGS! In AWS these are key-value pairs. On mine I put:

| Key  | Value           |
| ---- | --------------- |
| Name | Test-Web-Server |
| Role | Web-Server      |

But if you want you can get super granular with these to make your policies specific. For example you could have a key-value pair `ExternalAccessAllowed:True` or `location:NAM`

5. Launch the server

# Configuring the Plugin
When you're configuring the plugin through Panorama you don't need to manually enter all of these fields. It just uses its assigned IAM role. Because that doesn't work on the firewall we need to do it by hand.
1. Go to Device → VM Information Sources and Add
2. Name: AWS-Source
3. Type: AWS VPC
4. Source: enter the URI for your VPC `ec2.<your_region>.amazonaws.com`
5. Access Key ID: found in IAM → user → create access key (this will give you a new access key/private key pair)
6. Create it, commit it, and then the status should go green
# Creating our 1st DAG!
1. Go to Objects → Address Groups → Add
2. Name: DAG-AWS-WebServers
3. Type: Dynamic
4. Add match criteria and (if your poll interval has passed) you can see the tags from AWS populate ![DAGCreation.png](Assets/DAGCreation.png)
5. Now we can make policies based on tags, VPC IDs, Subnet IDs, etc and enforce them as soon as the server gets spun up!

# Auto tagging with threats
If you want to test the **Threat** engine specifically but safely, you can pick a very common, low-risk signature (like a ping) and tell the firewall to treat it as a "Threat" for just one rule.

1. Objects > Security Profiles > Vulnerability Protection: Create a new profile.
2. Exceptions: Search for a basic ICMP (Ping) signature ID and change its action to **Alert**.
3. Policy: Attach this profile to your "Allow" rule.
4. Test: Ping the firewall. It will generate a "Threat" log for the ping, which your Log Forwarding Profile will then use to trigger the auto-tagging.
