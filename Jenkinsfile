@Library('my-shared-library') _

pipeline{

    agent {
        docker {
          image 'abhishekf5/maven-abhishek-docker-agent:v1'
          args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // mount Docker socket to access the host's Docker daemon
        }
    }
//     environment {
//             PATH = "/usr/bin:$PATH"
//     }
//     tools {
//         maven 'mvn'
//     }

    parameters{
        choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
        string(name: 'ImageName', description: "name of the docker build", defaultValue: 'javapp')
        string(name: 'ImageTag', description: "tag of the docker build", defaultValue: 'v1')
        string(name: 'DockerHubUser', description: "name of the Application", defaultValue: 'trex1987')
        string(name: 'SonarQubecredentialsId', description: "name of the Application", defaultValue: 'sonarqube-api')
    }

    stages{
        stage('trivy test') {
            steps {
                echo 'trivy --version'
            }
        }
         
        stage('Git Checkout'){
            when {
                expression {
                    params.action == 'create'
                }
            }
            steps{
                gitCheckout(
                    branch: "main",
                    url: "https://github.com/trextozyne/Java_CI-CD_Sharelibrary.git"
                )
            }
        }

         stage('Unit Test maven'){
         
             when {
                   expression {
                        params.action == 'create'
                   }
             }

             steps {
                   script{
                       mvnTest()
                   }
             }
         }

         stage('Integration Test maven'){
         when { expression {  params.action == 'create' } }
            steps{
               script{
                   mvnIntegrationTest()
               }
            }
        }

        stage('Static code analysis: Sonarqube'){
            when {
               expression {
                   params.action == 'create'
               }
            }
            steps{
               script{

                   def SonarQubecredentialsId = "${params.SonarQubecredentialsId}"
                   staticCodeAnalysis(SonarQubecredentialsId)
               }
            }
        }

        // make sure to, sonarqube->administration->create(webhooks)->URL->http://server_ip:8080/sonarqube-webhook/
        stage('Quality Gate Status Check : Sonarqube'){
             when {
                 expression {
                     params.action == 'create'
                 }
             }

            steps{
                script{
                    def SonarQubecredentialsId = "${params.SonarQubecredentialsId}"
                    QualityGateStatus(SonarQubecredentialsId)
                }
            }
        }

        stage('Maven Build : maven'){
            when {
                expression {
                    params.action == 'create'
                }
            }

            steps{
               script {
                   mvnBuild()
               }
            }
        }

        stage('Docker Image Build'){
//          agent {  docker { image ''}  }
            when {
                expression {
                    params.action == 'create'
                }
            }

            steps{
               script{
                   dockerBuild("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
               }
            }
        }

        stage('Docker Image Scan: trivy '){
            when {
                expression {
                    params.action == 'create'
                }
            }

            steps{
               script{
                   dockerImageScan("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
               }
            }
       }

        stage('Docker Image Push : DockerHub '){
//          agent {  docker { image ''}  }
            when {
                expression {
                    params.action == 'create'
                }
            }

            steps{
               script{
                   dockerImagePush("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
               }
            }
        }

        stage('Docker Image Cleanup : DockerHub '){
//          agent {  docker { image ''}  }
            when {
                expression {
                    params.action == 'create'
                }
            }

            steps{
               script{
                   dockerImageCleanup("${params.ImageName}","${params.ImageTag}","${params.DockerHubUser}")
               }
            }
        }      
    }
}