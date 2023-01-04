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
                sh """sed -i 's|DOCKER_IMAGE_NAME|${env.DOCKER_IMAGE_NAME}|g' train-schedule-kube-canary.yml"""
                sh """sed -i 's|BUILD_NUMBER|${env.BUILD_NUMBER}|g' train-schedule-kube-canary.yml"""
                //sh "cat train-schedule-kube-canary.yml"
                withKubeConfig(credentialsId: 'kubernetes_auth', namespace: '', serverUrl: 'https://172.31.6.117:6443') {
                sh "kubectl apply -f train-schedule-kube-canary.yml"
                sh 'DEPLOYMENT_NAME=$(kubectl get deployments | grep canary | awk \'{print $1}\')'
                sh "kubectl scale --replicas=${env.CANARY_REPLICAS} ${DEPLOYMENT_NAME}"
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
                sh """sed -i 's|DOCKER_IMAGE_NAME|${env.DOCKER_IMAGE_NAME}|g' train-schedule-kube.yml"""
                sh """sed -i 's|BUILD_NUMBER|${env.BUILD_NUMBER}|g' train-schedule-kube.yml"""
                sh "cat train-schedule-kube.yml"
                withKubeConfig(credentialsId: 'kubernetes_auth', namespace: '', serverUrl: 'https://172.31.6.117:6443') {
                sh 'DEPLOYMENT_NAME=$(kubectl get deployments | grep canary | awk \'{print $1}\')'
                sh "kubectl scale --replicas=${env.CANARY_REPLICAS} ${DEPLOYMENT_NAME}"
                sh "kubectl apply -f train-schedule-kube.yml"
                }
            }
        }
    }
}
