## AWS-Ansible Lab

The files in this repository/directory are from a hands-on lab given at the Austin Ansible meetup in May 2015. This lab utilized AWS CloudFormation and Ansible to complete the following exercises:

### Lab Exercises:

Lab 1: Write code in a language or tool of your choice that provisions an environment running on the Linux or Windows operating system that hosts a static website with a single HTML page.

Lab 2: Extend the first lab to make the solution highly available.

Lab 3: Extend the second lab to allow for zero downtime updates of the static website using pre-baked AMIs.

Lab 4: Extend the second lab to allow for zero downtime updates of the static website using a bootstrap process.

Extra-credit 1: Demonstrate a roll-back of the static website from Lab 3.

Extra-credit 2: Demonstrate a roll-back of the static website from Lab 4.


The solutions were developed and tested using an EC2 Default VPC. They are not intended to work with in an EC2-Classic environment. Some changes are necessary for an EC2 VPC environment.

* Lab 1 solution: Single instance running Nginx and serving a single web page.
* Lab 2 solution: ELB in front of an AutoScaling Group with rolling updates.
* Lab 3 solution: Ansible managing the CloudFormation stack. This solution is more customizable that the Lab 1 and Lab 2 solution thus it has several more prompts for input. While the first two solutions use Amazon Linux, this solution uses Ubuntu 14.04.
* Lab 4 solution: Ansible managing the CloudFormation stack. This solution uses a different CloudFormation template that removes the rolling update process and replaces it with cfn-init pulling files from S3.



### Lab1.json

##### Pre-requisites:
* AWS Command Line Interface installed and permissions to create the necessary AWS resources (cloudformation:CreateStack, ec2:CreateSecurityGroup, etc.).

##### Command to run:
```shell
aws cloudformation create-stack \
 --stack-name myStackName \
 --template-body file://Lab1.json \
 --parameters ParameterKey=InstanceType,ParameterValue=t2.micro
```

##### Noteworthy Details:

- No security group rule allowing SSH access to instance. No KeyPair defined for the instance.
  
  These are needed for SSH access to an instance for logs, troubleshooting, etc. I believe once a proper setup is complete, SSH access is not needed and only provides a security risk. If necessary, there are ways to remove the instance from active service and gain SSH access to an instance without an initial KeyPair.
- Security group allow ingress to port 80 (http) and 443 (https) from anywhere.
  
  Opened http and https ports from anywhere because it is a web server. I would normally install and use a SSL certificate or close port 443 at the security group level.
- Using a WaitCondition
  
  Want all UserData steps to complete before notifying stack as complete. This helps ensure the web server is up and running when the stack is identified as complete.

### Lab2.json

##### Pre-requisites:
* AWS Command Line Interface installed, and permissions to create a CloudFormation stack.

##### Command to run:
```shell
aws cloudformation create-stack \
 --stack-name myStackName2 \
 --template-body file://Lab2.json \
 --parameters ParameterKey=InstanceType,ParameterValue=t2.micro \
              ParameterKey=VpcId,ParameterValue=vpc-173c2575 \
 --capabilities CAPABILITY_IAM
```

##### Noteworthy Details:
- Added an Elastic Load Balancer (ELB) and AutoScaling Group (ASG).

  My typically preferred "unit of deployment" is an ELB with an ASG behind it.
  URL points to the ELB. As a result, user traffic goes to the ELB which forwards the traffic on to the instances behind it. You can perform SSL termination at the ELB.
  Only the ELB accepts internet traffic. Only the ELB can initiate traffic with the instances. This is for security purposes.

- Used DependsOn to create single security group for ELB

    Without using DependsOn, CloudFormation will create a default security group for the ELB.


### Lab3.json
This solution creates AMIs with the static file included in the image.

By creating and using AMIs, we have greater flexibility in updating the machines running the code. Instead of deploying a single file or code artifact, we are deploying an entire machine image. This can provide advantages in the case a rollback is needed.

New AMIs are created for any code updates and the CloudFormation stack is updated with the new AMI. When the stack is updated, the running instances are replaced by using an AutoScaling Group Update Policy.

Two drawbacks of this solution are the additional costs incurred due to terminating instances and the length of time the rolling update process takes.

##### Pre-requisites:
* This solution is intended to be run in AWS Region US-West-2.
* This solution requires an S3 bucket with a zip archive of the static html to serve.

* Ansible control machine.

  I have included a sample template, ansibleForDefaultVpcOnly.json, that can be used to create an Ansible control machine in EC2 Default VPC. This template does require a SSH key file named ansible.pem to be located in an S3 bucket (passed as a parameter) under the folder ssh_keys.
  e.g., s3://mybucket/ssh_keys/ansible.pem

The command I used to create the stack (with user specific ParameterValues replaced with [blank]):

```shell
aws cloudformation create-stack \
 --stack-name myAnsibleStack \
 --template-body file://ansibleForDefaultVpcOnly.json \
 --parameters ParameterKey=AnsibleAMI,ParameterValue=ami-3389b803 \
              ParameterKey=AnsibleInstanceCountMax,ParameterValue=1 \
              ParameterKey=AnsibleInstanceCountMin,ParameterValue=1 \
              ParameterKey=AnsibleInstanceType,ParameterValue=t2.micro \
              ParameterKey=BucketName,ParameterValue=[blank] \
              ParameterKey=KeyName,ParameterValue=[blank] \
              ParameterKey=HomeNetwork0CIDR,ParameterValue=[blank] \
              ParameterKey=VpcId,ParameterValue=[blank] \
 --capabilities CAPABILITY_IAM
 ```

##### Commands to run:
From the Ansible control machine, clone the GitHub repository.
```shell
git clone https://github.com/joehack3r/ansibleMeetup201505.git
cd ansibleMeetup201505
```

Suggested: Edit the playbook file to provide your own defaults for the var_prompts items. To help, the security_group_id and iam_instance_profile and listed as outputs from myAnsibleStack.

Create an AMI and CloudFormation stack:
```shell
ansible-playbook Lab3.yml
  What is the Amazon Machine Image (AMI) ID to use (Default is: Ubuntu 14.04, HVM:EBS, amd64, 20150506) [ami-3389b803]: 
  What is the VPC Subnet ID to use: []: 
  What is the Amazon Security Group to use []: 
  What is the IAM Instance Profile to use []: 
  In which S3 bucket is the zipped static web file archive stored? []: 
  What is the path to the file? []:
  What is the name of the archive? [webfiles1.zip]: 
  What is the name of CloudFormation stack to create/update? [myStackName3]: 
  What is the VPC ID to launch the CloudFormation stack? []: 
```

### Lab4.json
This solution creates an AMIs that downloads the static file, passed as a parameter to CloudFormation, during the boot process. If the parameter value changes, the instance re-downloads the file. This provides another deployment option that can update running instances without terminating them.

Compared to deploying AMIs, this option lets the instance pull new code. If the AMI needs to be updated (e.g, security patching), 

Create an AMI and CloudFormation stack:
```shell
ansible-playbook Lab4.yml
```

##### Noteworthy Details:
- cfn-hup moved to later in the bootstrap process.

Still investigating why this was needed for Lab4.json and not Lab3.json.

- No notification of successful update.

Because the downloading of the file is done by the cfn-hup process, there is no verification process.


### Extra Credit 1
To demonstrate a roll-back of the Lab3 solution, update the stack with the original AMI:
```shell
aws cloudformation describe-stacks --stack-name myStackName3
aws cloudformation get-template --stack-name myStackName3 | jq .TemplateBody > foo3.json
aws cloudformation describe-stacks --stack-name myStackName3 | jq .Stacks[].Parameters > bar3.json
vi bar3.json
aws cloudformation update-stack --stack-name myStackName3 --template-body file://foo3.json --parameters file://bar3.json --capabilities CAPABILITY_IAM
```

### Extra Credit 2
To demonstrate the usefulness of the Lab4 solution, update the stack to use a different archive file name:
```shell
aws cloudformation describe-stacks --stack-name myStackName4
aws cloudformation get-template --stack-name myStackName4 | jq .TemplateBody > foo4.json
aws cloudformation describe-stacks --stack-name myStackName4 | jq .Stacks[].Parameters > bar4.json
vi bar4.json
aws cloudformation update-stack --stack-name myStackName4 --template-body file://foo4.json --parameters file://bar4.json --capabilities CAPABILITY_IAM
```
