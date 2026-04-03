pipeline {
    agent any
    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        SONARQUBE_ENV = 'sq'
        DOCKER_IMAGE = "rajeshtutta123/zomato"
        AWS_CREDS = credentials('aws-creds')
        AWS_DEFAULT_REGION = 'us-east-1'
        RECIPIENTS = 'rajeshtutta123@gmail.com'
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/rajeshtutta/zomato.git'
            }
        }
       stage('Install & Build') {
    steps {
        sh 'npm install && npm run build'
    }
}
      stage('Package Artifact') {
    steps {
        sh 'zip -r zomato-build.zip build/'
    }
}
stage('JENKINS TO NEXUS') {
    steps {
        withMaven(jdk: 'jdk21', traceability: true) {
            sh 'npm install'
            sh 'npm run build'
        }
    }
}
        stage('Upload to Nexus') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'nexus-cred',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
        )]) {
            sh '''
            curl -v -u $NEXUS_USER:$NEXUS_PASS \
            --upload-file zomato-build.zip \
            http:44.203.55.56:8081/repository/raw-hosted/zomato-build.zip
            '''
        }
    }
}
        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv("${SONARQUBE_ENV}") {
            sh '''
            npm install -g sonar-scanner
            sonar-scanner \
              -Dsonar.projectKey=zomato \
              -Dsonar.sources=src \
              -Dsonar.host.url=$SONAR_HOST_URL \
              -Dsonar.login=$SONAR_AUTH_TOKEN
            '''
        }
    }
}

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }


        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:latest .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_IMAGE:latest
                    docker logout
                    '''
                }
            }
        }
         stage('Build') {
            steps {
                echo "Building..."
            }
        }

        stage('Test') {
            steps {
                echo "Testing..."
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                export AWS_ACCESS_KEY_ID=$AWS_CREDS_USR
                export AWS_SECRET_ACCESS_KEY=$AWS_CREDS_PSW

                aws eks update-kubeconfig --region us-east-1 --name mycluster
                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }
    }

    post {
        success {
            emailext(
                subject: "Jenkins Job '${env.JOB_NAME}' Success",
                body: "Good news! Job '${env.JOB_NAME}' (#${env.BUILD_NUMBER}) succeeded.\n\nCheck console output at ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "Jenkins Job '${env.JOB_NAME}' Failed",
                body: "Alert! Job '${env.JOB_NAME}' (#${env.BUILD_NUMBER}) failed.\n\nCheck console output at ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }

    }
}
