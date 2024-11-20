pipeline {
    agent any

    tools{
        jdk 'JDK 17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = 'arunagri03/netflix'
    }

    stages{
        stage ('workspace clean') {
            steps{
                cleanWs()
            }
        }

        stage('git checkout'){
            steps{
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/arun037/netflix_2.git']])
            }
        }

        stage('sonarqube Analysis') {
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=netflix \
                    -Dsonar.projectKey=netflix '''

                }
            }
        }

        stage('Qualitygate analysis') {
            steps{
                waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
            }
        }

        stage('install dependencies') {
            steps{
                sh 'npm install'
            }
        }

         stage('OWASP FS SCAN') {
         steps {
             dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
             dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
         }
     }

        stage('trivy fs scan') {
            steps{
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('docker build and tag') {
            steps{
                 sh "docker build --build-arg TMDB_V3_API_KEY=0114d3a977ba9baf55c6f8b83d657853 -t netflix ."
                 sh 'docker tag netflix $IMAGE_TAG'
            }
        }

        stage('trivy image scan') {
            steps{
                sh 'trivy image $IMAGE_TAG -o trivy-image-report.html'
            }
        }

        stage('docker image push') {
            steps{
                 withDockerRegistry(url: 'https://index.docker.io/v1/', credentialsId: 'docker-token') {
                    sh 'docker push $IMAGE_TAG'
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withAWS(credentials: 'aws_id') {
                    script {
                        dir('Kubernetes') {
                            sh 'aws eks update-kubeconfig --region ap-south-1 --name my-cluster'
                            sh 'kubectl apply -f deployment.yml'
                            sh 'kubectl apply -f service.yml'
                            sh 'sleep 60'
                        }
                    }
                }
            }
        }
        
       stage ('EKS Verification') {
            steps {
                withAWS(credentials: 'aws_id') {
                    sh 'aws eks update-kubeconfig --region ap-south-1 --name my-cluster' 
                    sh 'kubectl get pods,deploy,svc -A'
                }
            }
        }
    }    
    
    post {
        always {
            emailext(
                attachLog: true,
                subject: "'Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}'",
                body: """<p>Project: ${env.JOB_NAME}</p>
                         <p>Build Number: ${env.BUILD_NUMBER}</p>
                         <p>URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>""",
                to: 'arunagri03@gmail.com',
                attachmentsPattern: 'trivy-fs-report.html,trivy-image-report.html'
            )
        }
    }
}



