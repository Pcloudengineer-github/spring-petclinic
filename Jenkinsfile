pipeline {
    agent any

    environment {
        // ECR Details - Replace with your values
        AWS_ACCOUNT_ID = "952875262382"
        AWS_DEFAULT_REGION = "ap-south-1"
        ECR_REPOSITORY = "petclinic"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        
        // App Details
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
    }

    stages {
        // =================================================================
        //                       BUILD PIPELINE (CI)
        // =================================================================
        stage('Clone Repository') {
            steps {
                git 'https://github.com/spring-projects/spring-petclinic.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') { // Matches SonarQube server name in Jenkins config
                    sh 'mvn sonar:sonar \
                        -Dsonar.projectKey=YourProjectKey \
                        -Dsonar.organization=YourOrgKey \
                        -Dsonar.host.url=https://sonarcloud.io \
                        -Dsonar.login=$SONAR_TOKEN'
                }
            }
        }
        
        stage('Build & Push Docker Image') {
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${build.number}"
                    def latestTag = "${IMAGE_NAME}:latest"

                    // Login, Build, and Push
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                    sh "docker build -t ${imageTag} ."
                    sh "docker tag ${imageTag} ${latestTag}"
                    sh "docker push ${imageTag}"
                    sh "docker push ${latestTag}"
                }
            }
        }
        
        stage('Trivy Vulnerability Scan') {
            steps {
                // Scan the image we just pushed
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${build.number}"
            }
        }

        // =================================================================
        //                       RELEASE PIPELINE (CD)
        // =================================================================
        stage('Deploy to EKS') {
            steps {
                script {
                    // Update deployment manifest with the correct image URI
                    def imageUri = "${IMAGE_NAME}:${build.number}"
                    sh "sed -i 's|AWS_ECR_URI_PLACEHOLDER|${imageUri}|' k8s/deployment.yaml"
                    
                    // Use Kubernetes CLI plugin with EKS config
                    withKubeConfig([credentialsId: 'aws-kubeconfig', serverUrl: '']) {
                        sh 'kubectl apply -f k8s/deployment.yaml'
                        sh 'kubectl apply -f k8s/service.yaml'
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh 'sleep 30' // Wait for pods to start
                sh 'kubectl rollout status deployment/petclinic-deployment'
                sh 'kubectl get service petclinic-service'
            }
        }
    }
}
