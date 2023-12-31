def imageName="dawidwoj/backend"
def dockerRegistry=""
def registryCredentials="Dockerhub"
def dockerTag=""

pipeline {
  agent {
    label 'agent'
    }
    environment {
      scannerHome = tool 'SonarQube'
    }
    stages{
      stage('Git pull'){ 
        steps {
         //  git branch: 'main', url: 'https://github.com/DawidWojda/Frontend'
         checkout scm
         }
      }
      stage('test'){
        steps {
            sh 'ls'
            sh 'pip3 install -r requirements.txt'
            sh 'python3 -m pytest --cov=. --cov-report xml:test-results/coverage.xml --junitxml=test-results/pytest-report.xml'
         }
      } 
      stage('SonarQube') {
        steps {
            withSonarQubeEnv('SonarQube') {
                sh "${scannerHome}/bin/sonar-scanner"
             }
             timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
             }
        }
      }
    stage('Build application image') {
        steps {
            script {
                  // Prepare basic image for application
                  dockerTag = "RC-${env.BUILD_ID}"
                  applicationImage = docker.build("$imageName:$dockerTag",".")
            }
          }
        }
        
        stage('Pushing image to Artifactory') {
            steps {
                script {
                    docker.withRegistry("$dockerRegistry", "$registryCredentials") {
                        applicationImage.push()
                        applicationImage.push('latest')
                    }
                }
            }
        }
stage ('Push to repo') {
            steps {
                dir('ArgoCD') {
                    withCredentials([gitUsernamePassword(credentialsId: 'git', gitToolName: 'Default')]) {
                        git branch: 'main', url: 'https://github.com/DawidWojda/ArgoCD'
                        sh """ cd backend
                        git config --global user.email "dawidwojdalf@gmail.com"
                        git config --global user.name "dawid"
                        sed -i "s#$imageName.*#$imageName:$dockerTag#g" deployment.yaml
                        git commit -am "Set new $dockerTag tag."
                        git diff
                        git push origin main
                        """
                    }                  
                } 
            }
        }
    }
    post {
        always {
            junit testResults: "test-results/*.xml"
            cleanWs()
        }
        success {
            build job: 'app_of_apps', parameters: [ string(name: 'backendDockerTag', value: "$dockerTag")], wait: false
        }
    }
}
