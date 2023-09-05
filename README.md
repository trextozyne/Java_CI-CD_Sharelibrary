# Jenkins Pipeline for Java based application using Maven, SonarQube, Argo CD, Helm and Kubernetes

![Screenshot 2023-03-28 at 9 38 09 PM](./ci-cd-eks.png)
## JAVA CI-CD pipeline
Jenkins


Install Jenkins, configure Docker as agent, set up cicd, deploy applications to k8s and much more.

## AWS EC2 Instance

- Go to AWS Console
- Instances(running)
- Launch instances

<img width="994" alt="Screenshot 2023-02-01 at 12 37 45 PM" src="https://user-images.githubusercontent.com/43399466/215974891-196abfe9-ace0-407b-abd2-adcffe218e3f.png">

### Install Jenkins.
In the EC2 userdata add scripts
1. [Install Jenkins](/home/the-rexy/OneDrive/Desktop/dev-projects/interview projects/installation_scripts.git/jenkins.sh)
2. [Install Docker](/home/the-rexy/OneDrive/Desktop/dev-projects/interview projects/installation_scripts.git/docker.sh)
3. [Install SonarQube](/home/the-rexy/OneDrive/Desktop/dev-projects/interview projects/installation_scripts.git/sonarqube.sh) as Docker container 
4. Then Launch Jenkins

### Login to Jenkins using the below URL:

http://<ec2-instance-public-ip-address>:8080    [You can get the ec2-instance-public-ip-address from your AWS EC2 console page]

Note: If you are not interested in allowing `All Traffic` to your EC2 instance
1. Delete the inbound traffic rule for your instance
2. Edit the inbound traffic rule to only allow custom TCP port `8080`

After you login to Jenkins,
- Run the command to copy the Jenkins Admin Password - `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
- Enter the Administrator password

<img width="1291" alt="Screenshot 2023-02-01 at 10 56 25 AM" src="https://user-images.githubusercontent.com/43399466/215959008-3ebca431-1f14-4d81-9f12-6bb232bfbee3.png">
- cat address location to get password

### Click on Install suggested plugins

<img width="1291" alt="Screenshot 2023-02-01 at 10 58 40 AM" src="https://user-images.githubusercontent.com/43399466/215959294-047eadef-7e64-4795-bd3b-b1efb0375988.png">

Wait for the Jenkins to Install suggested plugins

<img width="1291" alt="Screenshot 2023-02-01 at 10 59 31 AM" src="https://user-images.githubusercontent.com/43399466/215959398-344b5721-28ec-47a5-8908-b698e435608d.png">

Create First Admin User or Skip the step [If you want to use this Jenkins instance for future use-cases as well, better to create admin user]

<img width="990" alt="Screenshot 2023-02-01 at 11 02 09 AM" src="https://user-images.githubusercontent.com/43399466/215959757-403246c8-e739-4103-9265-6bdab418013e.png">

Jenkins Installation is Successful. You can now starting using the Jenkins

<img width="990" alt="Screenshot 2023-02-01 at 11 14 13 AM" src="https://user-images.githubusercontent.com/43399466/215961440-3f13f82b-61a2-4117-88bc-0da265a67fa7.png">



a. Troubleshoot, make sure Docker is running
i. It has 2 Jenkins files for pipeline that uses build with parameter using CHOICE(uses a dropbdown: when { expression {  params.action == 'create' } } and STRING(uses textboxt)
ii. pipeline for local/dockerhub repository and when run use build with parameter
```groovy
parameters{
choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
string(name: 'ImageName', description: "name of the docker build", defaultValue: 'javapp')
string(name: 'ImageTag', description: "tag of the docker build", defaultValue: 'v1')
string(name: 'DockerHubUser', description: "name of the Application", defaultValue: 'trex1987')
}
```

iii. pipeline for AWS ECR
```groovy
parameters{

    choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
    string(name: 'aws_account_id', description: " AWS Account ID", defaultValue: '690274493847')
    string(name: 'Region', description: "Region of ECR", defaultValue: 'us-east-2')
    string(name: 'ECR_REPO_NAME', description: "name of the ECR", defaultValue: 'the-rexy-java')
    string(name: 'cluster', description: "name of the EKS Cluster", defaultValue: 'demo-cluster1')
}
```
iv. CD part
We use: ```dir('eks_module') {…}```, to get access to terraform file
i. I want the terraform aws access keys to be called from Jenkins and not hard coded into terraform using, we makes sure we have terraform installed and configured on local system:
later add sh script dev, stage, QA, Production
```shell
environment{
ACCESS_KEY = credentials('AWS_ACCESS_KEY_ID')
SECRET_KEY = credentials('AWS_SECRET_KEY_ID')
}
```
Then call it in Jenkins Eks part as and run im multiline the terraform scripts:
```shell
dir('eks_module') {
sh """
terraform init
terraform plan -var 'access_key=$ACCESS_KEY' -var 'secret_key=$SECRET_KEY' -var 'region=${params.Region}' --var-file=./config/terraform.tfvars
terraform apply -var 'access_key=$ACCESS_KEY' -var 'secret_key=$SECRET_KEY' -var 'region=${params.Region}' --var-file=./config/terraform.tfvars --auto-approve
"""
}
```
ii. Next we make sure we install and configure aws-cli on server, 	set the Access & secret key  and connect to EKS using:
```shell
script{
sh """
aws configure set aws_access_key_id "$ACCESS_KEY"
aws configure set aws_secret_access_key "$SECRET_KEY"
aws configure set region "${params.Region}"
aws eks --region ${params.Region} update-kubeconfig --name ${params.cluster}
"""
}
```
iii. Next we deploy k8s manifest to EKS in Jenkins:
Using choice to get it started, then use “input message: “ to get true or false, it shows inline in pipeline or console view, if false don’t fail show red sight, just exit as unstable)
script{
```groovy
def apply = false

  try{
    input message: 'please confirm to deploy on eks', ok: 'Ready to apply the config ?'
    apply = true
  }catch(err){
    apply= false
    currentBuild.result  = 'UNSTABLE' //dont make the current build FAIL if user has deniewd
  }
  if(apply){
    sh """
      kubectl apply -f .
    """
  }
```

