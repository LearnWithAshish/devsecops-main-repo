pipeline {
    agent any

environment {
    imageName = "dsocouncil/node-service:${GIT_COMMIT}"
  }

    stages {
        stage('Build Artifact') {
            steps {
                sh "mvn clean package -DskipTests=true"
                archiveArtifacts artifacts: 'target/*.jar', onlyIfSuccessful: true
            }
        }
        stage('Check Git-Secrets') {
            steps {
                script {
                    // Define the GitHub repository URL
                    def githubRepoURL = 'https://github.com/shrivastavashish/devsecops-main-repo.git'

                    // Remove existing trufflehog file if it exists
                    sh "rm trufflehog || true"

                    // Run Trufflehog container to scan GitHub repository
                    sh """
                    docker run --rm -v \"$PWD:/pwd\" \
                    trufflesecurity/trufflehog:latest github --repo https://github.com/shrivastavashish/devsecops-main-repo.git --json > trufflehog_report.json
                """
                // copy file to server 
                    sh "sudo cp trufflehog_report.json /root/reports/trufflehog/"
                }
            }
        }
        stage('Static Analysis - SonarQube') {
            steps {
                script {
                    def sonarProjectKey = 'devsecops'
                    def sonarHostUrl = 'http://54.89.224.127:9000/'
                    def sonarToken = 'sqp_a75e8fe4cf8f67f1bada216f7e5d3c799c893a32'

                    withSonarQubeEnv('devsecops') {
                        sh "mvn sonar:sonar -Dsonar.projectKey=${sonarProjectKey} -Dsonar.host.url=${sonarHostUrl} -Dsonar.login=${sonarToken}"
                    }
                }

        stage('SCA Scan - Dependency-Check') {
            steps {
                sh "mvn dependency-check:check"
            }
            post {
                always {
                    dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                }
            }
        }

//         stage('Snyk Code Scan') {
//             steps {
//                 script {
//             // Set the SNYK_TOKEN environment variable
//             withEnv(["SNYK_TOKEN=56355cf5-fcf9-4a2a-91d6-50057a2e8038"]) {
//                 echo 'Testing...'
//                 snykSecurity(
//                     snykInstallation: 'snyk',
//                     snykTokenId: 'snyk',
//                     // Place other parameters here as needed
//                 )
//             }
//         }
//     }
// }
        stage('Trivy Scan') {
                    steps {
                        script {
                            // Run Trivy for vulnerability scanning
                            sh "bash trivy-scan.sh"
                        } 
                    }
                }
        stage('Docker Build and Push') {
            steps {
                script {
                    def dockerImageName = "dsocouncil/node-service:${env.GIT_COMMIT}"

                    withDockerRegistry(credentialsId: "dockerhub", url: "https://index.docker.io/v1/") {
                        sh "sudo docker build -t ${dockerImageName} ."
                        sh "docker push ${dockerImageName}"
                    }
                }
            }
        }
        stage('Kubernetes - Vulnerability Scan') {
      steps {
        parallel(
          "Kubesec Scan": {
            sh "bash kubesec-scan.sh"
          },
          "Trivy Scan": {
            sh "bash trivy-kuber-scan.sh"
          }
        )
      }
    }

        stage('Kubernetes- CIS Benchmark') {
            steps {
                script {

                parallel(
                    "Master": {
                    sh "bash cis-master.sh"
                    },
                    "Etcd": {
                    sh "bash cis-etcd.sh"
                    },
                    "Kubelet": {
                    sh "bash cis-kubelet.sh"
                    }
                )

                }
            }
            }    
        stage('Kubernetes Deployment - DEV') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "cp k8s_deployment_service.yaml k8s_deployment_service_temp.yaml"
                    sh "sed -i 's#replace#dsocouncil/node-service:${GIT_COMMIT}#g' k8s_deployment_service_temp.yaml"
                    sh "kubectl apply -f k8s_deployment_service_temp.yaml"
                    sh "rm k8s_deployment_service_temp.yaml"
                }
            }
        }