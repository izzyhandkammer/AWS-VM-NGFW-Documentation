# Notes on Accessing After Setup
- I could not access my management console with global protect enabled
- The IP assignment for the security groups is not dynamic - you need to change from "Custom" to "My IP" before each connection attempt
- I could not access over the corp network

# Setup and Prerequisites
- Request an AWS account through qtorque
## First Steps:
First we need to create an IAM policy. This is what is going to give the firewall permission to "talk" to the AWS API.
This isn't strictly required for basic stuff (non-HA deployments) but  you need it for HA, VM monitoring (polling for changes ie. using DAGs), and autoscaling. Since this is the point of the project go ahead and create a new policy:
- go to IAM → Policies → create a new policy
	``` 
	{
  "Version":"2012-10-17",
    "Statement":[      
       {
          "Sid":"VisualEditor0",         
          "Effect":"Allow",         
          "Action":[            
             "ec2:AttachNetworkInterface",            
             "ec2:DetachNetworkInterface",            
             "ec2:DescribeInstances",            
             "ec2:DescribeNetworkInterfaces"         
                   ],         
           "Resource":"*"
         }   
     ]
  }
  ```

`name: Palo-HA-Role`


# Networking Foundations
## Create VPC
This is the equivalent to an Azure VNet. 
- Go to the VPC Dashboard
- Create VPC
- Select **VPC only**
- Name it
- IPv4 CIDR: 10.0.0.0/16
## Create Subnets
- Create 3 subnets in the same Availability Zone

| Name       | IPv4 CIDR     | Availability Zone |
| ---------- | ------------- | ----------------- |
| Management | `10.0.1.0/24` | us-west-1a        |
| Untrust    | `10.0.2.0/24` | us-west-1a        |
| Trust      | `10.0.3.0/24` | us-west-1a        |
- Add the Internet Gateway via VPC → Internet Gateways → create
	- Name: Palo-Security-VPC
	- After creating, choose Actions → Attach to VPC and select your VPC
## Configure Route Tables
- Go to route tables. There will be a "Main" route already there. rename it to RT-Trust
- Create another one called RT-Public. This will be for the management and untrust subnets
- Associate the subnets
	- Select RT-Public → Subnet associations → Edit and associate the management and untrust subnets
	- Add the trust subnet to RT-Trust
- Add internet route to Public RT
	- Select RT-Public → Routes → Edit routes
	- Add route: Destination `0.0.0.0/0`, Target Internet Gateway → Palo-VPC-IGW

## Checklist
- One VPC
- Three subnets in the same AZ
- One internet Gateway attached
- Management and Untrust subnets point to IGW
- Trust subnets have NO routes yet

# Security Groups
Security Groups are basically virtual firewalls that are controlling our virtual resources. These are just port, protocol, IP rules, so we still have a job because the real traffic control happens IN the firewall.
We have to create 3 security groups, one for each interface
## Management SG
- Go to Security → security groups → create security group
- Name: SG-PA-Mgmt
- Description (required): Protects the Management interface
- VPC: Palo-Security-VPC
- Define inbound rules
	![[MGMT-Inbound.png]]
- Leave all traffic for outbound

## SG-Untrust
- Since we are going to do our own traffic inspection on the traffic, we can keep this open but controlled
- Inbound rules: 
	- All traffic
	- Source: `0.0.0.0/0`
## SG-Trust
- Inbound rules: 
	- All traffic
	- Source `10.0.0.0/16`

# Create ENIs
An ENI is a Virtual Network Card. So if the EC2 instance is the firewall, the ENIs are the physical ports.
In AWS, these exist independently of the VM. So we create an ENI, configure it, and then "plug it in" to the VM. Then if that VM dies, we can "unplug" it and move it to a new VM, with all the IP and security settings staying the same
## Creating them
- Go to the EC2 dashboard and look for network interfaces
- create three interfaces

| Name           | Subnet     | Security Group |
| -------------- | ---------- | -------------- |
| PA-Mgmt-ENI    | Management | SG-PA-Mgmt     |
| PA-Untrust-ENI | Untrust    | SG-Untrust     |
| PA-Trust-ENI   | Trust      | SG-Trust       |

## Crucial step
AWS has a safety feature that stops instances from passing traffic that isn't meant for _their_ IP. Since the whole point of a firewall is to pass traffic for other people, we need to disable this
- Select PA-Untrust-ENI
- Click Actions → change source/dest check
- Disable it
- Repeat for Trust (not management)

# Launching VM
We have VPCs, Subnets, Security Groups, and ENIs! Now we can launch the actual engine
## Launch the VM-Series Instance
- Go to the EC2 Dashboard → Instances → Launch Instances
- Name it what you want
- Choose the image from the marketplace. I chose VM-Series Next-Gen Virtual Firewall w/ Advanced Security Subs (PAYG) because go big or go home
- For the instance type i chose `c5.2xlarge`. I think whatever is cool just don't use a t series because apparently they don't work
- You have to create a key pair to control access to the instance. I chose RSA and `.pem` format. Make sure you know where you're keeping this file
## Connect the Virtual Cards to the Server
- Edit the network to be your Palo-Security-VPC
- Don't use the "Create Security Group" settings because if you do, it'll apply to all 3 of the interfaces → check "Select existing security group"
- When adding the ENIs you need to add them in the EXACT order thet PAN-OS expects them
	- Network Interface 1 (Device Index 0):
		- Click Network Interface → Select your PA-Mgmt-ENI. (When I did this it just showed the network interface ID so I pulled it up on another tab to cross check for peace of mind)
		- This will be the Management interface
	- Add network interface (Device index 1):
		- Select the PA-Untrust-ENI
		- This will be ethernet1/1
	- Add network interface (Device index 2):
		- Select your PA-Trust-ENI
		- This will be ethernet1/2
- once thats all done you can go ahead and launch

# Accessing the firewall
 We need to first allocate and associate the elastic IP. Elastic IPs (EIPs) are basically IPs you're "buying" from AWS that stay associated with your account. That means you can remap it to another instance.
- Go to VPC Dashboard → Elastic IPs
- Allocate an elastic IP address
- Select the new IP and click actions → select Associate Elastic IP address
- Resource type: Select Network interface
- Network interface: Select your PA-Mgmt-ENI
- Select the private IP address from the dropdown
- Verify that it was associated in the firewall
- Repeat this process for the untrust ENI so that the firewall has a public IP for the internet traffic
- Now wait around 10 mins for the services to finish setting up
Here's where that `.pem` file from earlier is going to be used.
- Once you've waited for the firewall to spin up, open terminal and connect via SSH:
	- First, change the file permissions of the `.pem` so only you can read it: `chmod 400 <your-key>.pem`
	- Then SSH in: `ssh -i "your-key.pem" admin@<your-new-public-ip>`. 
	- I had an issue here where the SSH was timing out. This means theres an issue with your security group for the management interface. Double check that its still accepting ssh traffic from your IP
- Now you should be in your firewall!
- Configure and set your password:
	- `configure`, `set mgt-config users admin password`, `commit`, `exit`
- Now you should be able to access your web GUI

# Bringing it to life
So now obviously the firewall is running but its not actually doing anything yet because the interfaces are unconfigured.
## Configure Interfaces
* Select ethernet1/1
	* Interface type: Layer3
	* Config tab:
		* Virtual router: default
		* Security zone: new zone. Name it the Untrust zone
	* IPv4 tab:
		* Select DHCP client
		* Critical: uncheck "Automatically create default route pointing to default gateway provided by server" (we want to control our own routing)
* ethernet1/2
	* Interface type: Layer3
	* Config tab:
		* Virtual router: default
		* Security zone: new zone. Name it Trust
	* IPv4:
		* DHCP client
	* Advanced tab (optional):
		* Click Interface Management Profile dropdown and create one called `Allow-Ping` and, yup, allow ping
## Create default static route
Because we unchecked "automatically create default route" we need to tell the fw how to reach the internet manually.
1. Go to Network → virtual routers and select default
2. Add a static route
	- Name: AWS-Default-Route
	- Destination: 0.0.0.0/0
	- Interface: ethernet1/1
	- Next hop: The first usable IP address of the Untrust Subnet
3. Commit!
4. Now both the ethernet interfaces should have a green up status
