pipeline {
  agent any

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
                sh "mvn test"
              },
              "Trivy Scan": {
                sh "bash trivy-docker-image-scan.sh"
              },
              "OPA Conftest": {
                sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-conftest-dockerfile.rego Dockerfile'
              }
            )
          }
        }

//        stage('Vulnerability Scan - Docker ') {
//          steps {
//            sh "mvn dependency-check:check"
//          }
//          post {
//            always {
//              dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
//            }
//          }
//        }

        stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "dockerhubcred", url: ""]) {
              sh 'printenv'
              sh 'sudo docker build -t alibr1996/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push alibr1996/numeric-app:""$GIT_COMMIT""'
            }
          }
        }

        stage('Kubernetes Deployment - DEV') {
          steps {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "sed -i 's#replace#alibr1996/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
              sh "kubectl apply -f k8s_deployment_service.yaml"
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
      }
    }

}