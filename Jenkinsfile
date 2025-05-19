def COLOR_MAP = [
    'FAILURE' : 'danger',
    'SUCCESS' : 'good'
]
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node24'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
                sh 'docker system prune -af --volumes'
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/xyan-dhgb/youtube-clone-app.git'
            }
        }

        stage('Install ESLint Plugin') {
            steps {
                sh 'npm install eslint-plugin-react --save-dev'
                }
        }

        stage('NPM Clean & Audit Fix') {
            steps {
                sh '''
            echo "Cleaning node_modules and package-lock.json"
            rm -rf node_modules package-lock.json

            echo "Installing fresh dependencies"
            npm install

            echo "Running npm audit fix"
            npm audit fix || true

            echo "Running npm audit fix --force"
            npm audit fix --force || true
        '''
            }
        }

        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=youtube \
                    -Dsonar.projectKey=youtube '''
                }
            }
        }   
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonarqube'
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }


        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'Docker', toolName: 'Docker'){
                       sh "docker build --build-arg REACT_APP_RAPID_API_KEY='9103301531mshe3656ce4ec9efd6p13e7d7jsnf408cd9a3d1b' -t youtube ."
                       sh "docker tag youtube xyan-dhgb/youtube:latest "
                       sh "docker push xyan-dhgb/youtube:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image --scanners vuln xyan-dhgb/youtube:latest > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name youtube1 -p 3000:3000 xyan-dhgb/youtube:latest'
            }
        }
        /*
        stage('Deploy to kubernets'){
            steps{
                withAWS(credentials: 'aws-key', region: 'us-east-1') {
                script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        sh 'kubectl apply -f deployment.yml'
                    }
                }
            }   }
        }
        */
    }
    post {
    always {
        echo 'Slack Notifications'
        slackSend (
            channel: '#all-youtube-clone-app',
            color: COLOR_MAP[currentBuild.currentResult],
            message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} \n build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        )
    }
}
}
