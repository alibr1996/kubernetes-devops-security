pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "alibr1996/numeric-app:${GIT_COMMIT}"
    applicationURL = "http://devsecops.eastus.cloudapp.azure.com/"
    applicationURI = "/increment/99"
  }

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true" //hello
              archive 'target/*.jar' //so that they can be downloaded later
            }
        }
        stage('Unit Tests - JUnit and Jacoco') {
              steps {
                sh "mvn test"
              }
        }

        stage('Mutation Tests - PIT') {
          steps {
            sh "mvn org.pitest:pitest-maven:mutationCoverage"
          }
        }

        stage('SonarQube - SAST') {
          steps {
            withSonarQubeEnv('sonarqube-server') {
                sh "mvn sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.host.url=http://devsecops.eastus.cloudapp.azure.com:9000" 
            }
            timeout(time: 2, unit: 'MINUTES') {
              script {
                waitForQualityGate abortPipeline: true
              }
            }
          }
        }

        stage('Vulnerability Scan - Docker') {
          steps {
            parallel(
              "Dependency Scan": {
                sh "mvn dependency-check:check"
              },
              "Trivy Scan": {
                sh "bash trivy-docker-image-scan.sh"
              },
              "OPA Conftest": {
                sh 'sudo docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-conftest-dockerfile.rego Dockerfile'
              }
            )
          }
        }


        stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "dockerhubcred", url: ""]) {
              sh 'printenv'
              sh 'sudo docker build -t alibr1996/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push alibr1996/numeric-app:""$GIT_COMMIT""'
            }
          }
        }

        stage('Vulnerability Scan - Kubernetes') {
          steps {
            parallel(
              "OPA Scan": {
                sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
              },
              "Kubesec Scan": {
                sh "bash kubesec-scan.sh"
              }
             // ,
             // "Trivy Scan": {
             //   sh "bash trivy-k8s-scan.sh"
             // }
            )
          }
        }

        stage('K8S Deployment - DEV') {
          steps {
            parallel(
              "Deployment": {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                  sh "bash k8s-deployment.sh"
                }
              },
              "Rollout Status": {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                  sh "bash k8s-deployment-rollout-status.sh"
                }
              }
            )
          }
        }


//        stage('Kubernetes Deployment - DEV') {
//          steps {
//            withKubeConfig([credentialsId: 'kubeconfig']) {
//              sh "sed -i 's#replace#alibr1996/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
//              sh "kubectl apply -f k8s_deployment_service.yaml"
//            }
//          }
//        }
        stage("DAST scanning- OWASP ZAP") {
          steps {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash zap.sh"
            }
          }
        } 

    }

    post {
      always {
        junit 'target/surefire-reports/*.xml'
        jacoco execPattern: 'target/jacoco.exec'
        pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
        publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'HTML Report', reportTitles: 'ZAP REPORT', useWrapperFileDirectly: true])
      }
    }

}