# onica-final
Take home test final for Onica Jr Devops position
The only pertinent file in this repo is the onica-final.json . I tested this with the user you have provided for me and it works as expected! 


# Design Considerations
I was tasked with creating a vpc with the following requirements:

1) VPC with Private/Public subnets and all required dependent infrastructure.

This was achieved in the following manner:
1 Internet Gateway associated with the VPC
2 Public subnets - Subnet 1 holds a bastion server in order to access the scaled instances that get created in the private subnets. Subnet 2 holds a NAT Gateway to allow outgoing traffic from the instances in the private subnets. These subnets are spread over 2 availability zones picked when launching the cloudformation template. 
2 Private subnets - these are used to house the instances for the auto scaling group. The private subnets use the same two availability zones so that I can set up the auto scaling group.
2 Route Tables - route table 1 is public and associated with public subnets 1 and 2. Contains one route to the IGW. route table 2 is private and associated with private subnets 3 and 4. Contains one route to the NAT gateway.

2) ELB used to register web server instances.

This was achieved in the following manner:
A load balancer is created associated with public subnets 1 and 2. This ELB has cross zone balancing enabled and will bounce requests back and forth depending on ASG (auto scaling group) availability. Listeners are set up on port 80. 

3) Autoscaling Group and Launch Configuration that launches EC2 images and registers them to the ELB

This was achieved in the following manner:
The autoscalinggroup is configured for a minimum of 2 and maximum of 5 instances. The instances are spun up in private subnets 3 and 4. The ASG uses a LaunchConfiguration called "OnicaWebASGLaunchConfig" which chooses an image appropriate for the instance type picked at startup. All instances are created associated with the ELB described above. The launchconfig installs apache and creates the startup page via a little script using UserData. The scaling policy is a TargetTrackingScaling policy type monitoring for average CPU utilization to be at 75%. 

4) Security group allowing http traffic to load balancer from anywhere

This was achieved in the following manner:
The OnicaELBSG security group sets this rule. It just adds an ingress rule for port 80. The web instances themselves have no public facing IP address and thus are not reachable from the internet except through the ELB or within the VPC from the bastion server.

5) Security group allowing http traffic only from the load balancer to the auto scaling group

This was achieved in the following manner:

The OnicaASGSG sets this rule. However, due to the lack of public IPs, the auto scaling group instances are not reachable from outside the network. However, to meet this requirement I created a ASG security group that explicitly created a rule using the SourceSecurityGroupId of the ELB:

`SourceSecurityGroupId" : { "Ref" :  "OnicaELBASB" }`

I also created a rule that only allows SSH traffic from the bastion server:

`"SourceSecurityGroupId" : { "Ref" : "OnicaBastionSG"} }`

Finally, I needed to add a `DependsOn : "OnicaELBSG"` line to ensure that the ASGSG doesn't start provisioning until the elastic load balancer is all set up.

6) Remote management ports such as ssh and rdp must not be open to the world.

This was achieved in the following manner:
The ports are only available within the network due to the network architecture.

7) Some kind of automation or scripting that achieves the following:
- Install a web server:
- Deploy a simple hello world page in the language of your choice
- Must include the server hostname in the page

This was accomplished by:
I kept this simple and created a bash script via the UserData parameter of the ASG launchconfiguration. THe bash script installs apache:
`yum install -y httpd`
Starts the apache service:
`service httpd start`
Creates a simple webpage using echo + a curl to http://169.254.169.254/latest/meta-data/hostname
`             "touch /var/www/html/index.html\n",
             "echo '<h2>Welcome to instance: </h2><p>' > /var/www/html/index.html\n",
             "curl http://169.254.169.254/latest/meta-data/hostname >> /var/www/html/index.html\n",
             "echo '</p>' >> /var/www/html/index.html\n" `
 
 8) All AWS resources must be created using CloudFormation or Teraform
 
 This was accomplished by:
I used cloudformation to do this. My major sticking point has been trying to get the aws-cfn-bootstrap tools installed to actually create files on the ASG instances. I ended up backing out of this method for the time being and will revisit it in my improvements section.

9) No resources may be created by hand except the EC2 SSH keys

This was accomplished by:
Pretty much goes with the above. You'll need to create teh keys via console or cli. Same key works across the entire vpc. 

Improvements:

Get that security group ingress policy figured out. I'm not certain how best to reference the groupid that I need. Maybe there is a getatt that'll get it done?

File management via aws-cfn-bootstrap and the helper scripts. I could not get the metadata to read correctly throughout the entire process. I know the bootstrap process starts, and I have cfn-init when the instance boots up, but its not installing packages listed in the packages section of the metadata. My package install is clunky and I feel like cfn scripts are closer to best practice. I'll need to hammer that out soon.

route1 takes a long time to create. I have a dependency on the nat gateway which holds it up. I don't know what takes that NAT gateway so long to spin up though. 

It'd be nice to make a slimmer AMI. These instances take a while to spin up and all I really want on them is apache.

The output should have the load balancer link
