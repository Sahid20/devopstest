# Lift & Shift Application Workflow to AWS
### Prerequisites:

- &nbsp; AWS Account 

- &nbsp; Registered DNS Name

- &nbsp; Maven

- &nbsp; JDK8

- &nbsp; AWS CLI
### Flow of Execution
- &nbsp; Login to AWS Account
- &nbsp; Create Key Pairs
- &nbsp; Create Security Groups
- &nbsp; Launch Instances with user data (bash scripts)
- &nbsp; Update IP to name mapping route 53
- &nbsp; Build Application from Source Code
- &nbsp; Upload to S3 bucket
- &nbsp; Download Artifact to Tomcat Ec2 Instance
- &nbsp; Setup ELB with HTTPS (Cert from Amazon Certificate Manager)
- &nbsp; Map ELB Endpoint to website name IONOS
- &nbsp; Build Autoscaling Group for Tomcat Instances
### Architecture on DataCenter:

![architecture_data_centre](https://user-images.githubusercontent.com/73986565/210881651-6abe39f1-a972-47ea-ac37-8e788ca82996.png)

### Architecture on AWS:

![aws_architecture](https://user-images.githubusercontent.com/73986565/210882506-d4ff2b00-29f5-4e06-9644-a740ba3b7ffb.png)

### Step-1: Create Security Groups for Services
- &nbsp; We will create vprofile-ELB-SG first. We will configure Inbound rules to Allow both HTTP and HTTPS on port 80 and 443 respectively from Anywhere IPv4 and IPv6.
![ELB](https://user-images.githubusercontent.com/73986565/210899681-63fbc4cf-3923-43e4-8b11-7cf7559b2aba.PNG)
Inbound rules
![ELB_inbound_rules](https://user-images.githubusercontent.com/73986565/210899758-90c8f0b1-9b2b-4cdf-b227-7f608046e4ac.PNG)
- &nbsp; Next we will create vprofile-app-SG. We will open port 8080 to accept connections from vprofile-ELb-SG
![vprofile_app](https://user-images.githubusercontent.com/73986565/210899966-8d9b8af0-6d1d-4ab8-8468-1a60033c896e.PNG)
Inbound Rules
![vprofile_app_inbound_rules](https://user-images.githubusercontent.com/73986565/210900042-10a28564-f862-4e22-baaf-01a231637d69.PNG)

- &nbsp; Finally, we will create vprofile-backend-SG. WE need to open port 3306 for MySQL, 11211 for Memcached and 5672 for RabbitMQ server. We can check whcih ports needed fro aplication services to communicate each other from application.properties file under src/main/resources directory.We also need to open commucation AllTraffic from own SecGrp for backend services to communicate with each other.
![vprofile_backens_sg_security_group](https://user-images.githubusercontent.com/73986565/210900113-4aded4ac-98a4-4689-8bdf-6561d749390c.PNG)
Inbound Rules
![vprofile_backens_sg_security_group_inbound_rules](https://user-images.githubusercontent.com/73986565/210900197-b9f2cdb4-761e-4b21-ae14-161926008743.PNG)

### Step-2: Create KeyPair to Connect EC2 instancesWe 
- &nbsp; Now, I will create a Keypair to connect our instances via SSH.
![key_pair](https://user-images.githubusercontent.com/73986565/210902132-a44f8a4a-7947-46c6-9820-036dc1500974.PNG)
### Step-3: Provision Backend EC2 instances with UserData script
#### DB Instance:
- &nbsp;Create DB instance with below details.We will also add Inbound rule to vprofile-backend-SG for SSH on port 22 from MyIP to be able to connect our db instance via SSH. <br>
Name: vprofile-db01 <br>
Project: vprofile <br>
AMI: Centos 7 <br>
InstanceType: t2.micro <br>
SecGrp: vprofile-backend-SG <br>
UserData: mysql.sh <br>
- &nbsp;Once our instance is ready, we can SSH into the server and check if userdata script is executed.We can also check status of mariadb <br>
ssh -i vprofile-prod-key.pem centos@<public_ip_of_instance> <br>
sudo -i  <br>
curl http://169.254.169.254/latest/user-data <br>
systemctl status mariadb <br>Memcached Instance:
#### Create Memcached instance with below details.
Name: vprofile-mc01
Project: vprofile
AMI: Centos 7
InstanceType: t2.micro
SecGrp: vprofile-backend-SG
UserData: memcache.sh
Once our instance is ready, we can SSH into the server and check if userdata script is executed.We can also check status of memcache service and if it is listening on port 11211.
ssh -i vprofile-prod-key.pem centos@<public_ip_of_instance>
sudo su -
curl http://169.254.169.254/latest/user-data
systemctl status memcached.service
ss -tunpl | grep 11211
#### RabbitMQ Instance:
Create RabbitMQ instance with below details.
Name: vprofile-rmq01
Project: vprofile
AMI: Centos 7
InstanceType: t2.micro
SecGrp: vprofile-backend-SG
UserData: rabbitmq.sh
Once our instance is ready, we can SSH into the server and check if userdata script is executed.We can also check status of rabbitmq service.
ssh -i vprofile-prod-key.pem centos@<public_ip_of_instance>
sudo su -
curl http://169.254.169.254/latest/user-data
systemctl status rabbitmq-server
Note: It may take some time to run userdata script after you connect to server. You can check the process ps -ef to see if the process start for service. If not wait sometime and check with systemctl status <service_name> command again

### Step-4: Create Private Hosted Zone in Route53
Our backend stack is running. Next we will update Private IP of our backend services in Route53 Private DNS Zone.Lets note down Private IP addresses.
rmq01 172.31.80.20
db01 172.31.22.178
mc01 172.31.87.132
Create vprofile.in Private Hosted zone in Route53. we will pick Default VPC in N.Virginia region
- &nbsp;Now we will create records for our backend services. The purpose of this activity is we will use these record names in our application.properties file. Even if IP address of the services, our application won't need to change the config file.
Simple Routing -> Define Simple Record
Value/Route traffic to: IP address or another value

### Step-5: Provision Application EC2 instances with UserData script
- &nbsp;Clone the repository.

Before we create our artifact, we need to do changes to our application.properties file under /src/main/resources directory for below lines.
jdbc.url=jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&

memcached.active.host=mc01.vprofile.in

rabbitmq.address=rmq01.vprofile.in
We will go to vprofile-project root directory to the same level pom.xml exists. Then we will execute below command to create our artifact vprofile-v2.war:
mvn install

### Step-6: Create Artifact Locally with MAVEN
- &nbsp;We will upload our artifact to s3 bucket from AWS CLI and our Tomcat server will get the same artifact from s3 bucket.

- &nbsp; We will create an IAM user for authentication to be used from AWS CLI.<br>

name: vprofile-s3-admin<br>
Access key - Programmatic access<br>
Policy: s3FullAccess<br>

### Step-7: Create S3 bucket using AWS CLI, copy artifact
- &nbsp;
### Step-8: Download Artifact to Tomcat server from S3
- &nbsp;
### Step-9: Setup LoadBalancer
- &nbsp;
### Step-10: Create Route53 record for ELB endpoint
- &nbsp;
### Step-11: Configure AutoScaling Group for Application Instances
- &nbsp;
### Step-12: Clean-up
- &nbsp;


