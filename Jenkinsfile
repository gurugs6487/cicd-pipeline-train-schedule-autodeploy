pipeline {
    agent any
    environment {
        //be sure to replace "bhavukm" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "ggsdocks/train-schedule"
        JAVA_HOME = '/usr/lib/jvm/java-8-openjdk-amd64'
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                    }
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {           
                withKubeConfig(credentialsId: 'kubernetes_auth', namespace: '', serverUrl: 'https://172.31.6.117:6443') {
                sh '''sed -i \'s/$CANARY_REPLICAS/CANARY_REPLICAS/g\' train-schedule-kube-canary.yml
                    kubectl apply -f train-schedule-kube-canary.yml --validate=false'''
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withKubeConfig(credentialsId: 'kubernetes_auth', namespace: '', serverUrl: 'https://172.31.6.117:6443') {
                sh '''sed -i \'s/$CANARY_REPLICAS/CANARY_REPLICAS/g\' train-schedule-kube-canary.yml
                    kubectl apply -f train-schedule-kube-canary.yml --validate=false
                    kubectl apply -f train-schedule-kube.yml'''
                }
            }
        }
    }
}
