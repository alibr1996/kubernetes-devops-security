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
              post {
                always {
                  junit 'target/surefire-reports/*.xml'
                  jacoco execPattern: 'target/jacoco.exec'
                }
              }
        }
        stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "dockerhubcred", url: ""]) {
              sh 'printenv'
              sh 'docker build -t siddharth67/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push siddharth67/numeric-app:""$GIT_COMMIT""'
            }
          }
        }

    }
}