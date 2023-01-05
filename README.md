# Lift & Shift Application Workflow to AWS
## Prerequisites:

- &nbsp; AWS Account 

- &nbsp; Registered DNS Name

- &nbsp; Maven

- &nbsp; JDK8

- &nbsp; AWS CLI
### Flow of Execution
- &nbsp 1; Login to AWS Account

### Architecture on DataCenter:

![architecture_data_centre](https://user-images.githubusercontent.com/73986565/210881651-6abe39f1-a972-47ea-ac37-8e788ca82996.png)

### Architecture on AWS:

![aws_architecture](https://user-images.githubusercontent.com/73986565/210882506-d4ff2b00-29f5-4e06-9644-a740ba3b7ffb.png)

### Flow of Execution
- &nbsp 1; Login to AWS Account
- &nbsp 1; Create Key Pairs
- &nbsp 1; Create Security Groups
- &nbsp 1; Launch Instances with user data (bash scripts)
- &nbsp 1; Update IP to name mapping route 53
- &nbsp 1; Build Application from Source Code
- &nbsp 1; Upload to S3 bucket
- &nbsp 1; Download Artifact to Tomcat Ec2 Instance
- &nbsp 1; Setup ELB with HTTPS (Cert from Amazon Certificate Manager)
- &nbsp 1; Map ELB Endpoint to website name IONOS
