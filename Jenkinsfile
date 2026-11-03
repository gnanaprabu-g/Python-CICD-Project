pipeline {
    agent any

     environment {
        REPO_URL = 'https://github.com/gnanaprabu-g/Python-CICD-Project.git'
        BRANCH = 'main'
        SQ_PROJECT_KEY = 'python-app'
        IMAGE_NAME = "python-app:${BUILD_NUMBER}"
        AWS_ACCOUNT_ID = '978355780378'
        AWS_REGION = 'ap-south-1'
        ECR_REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/python-app"
        ECR_IMAGE = "${ECR_REPO_URI}:${BUILD_NUMBER}"
        EKS_CLUSTER = 'demo-cluster-1'
    }

    stages {
        stage('Pull Code From GitHub') {
            steps {
                git branch: "${BRANCH}",
                    url: "${REPO_URL}"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SQ_PROJECT_KEY} \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                // script {
                //     env.IMAGE_NAME = "python-app:${BUILD_NUMBER}"
                // }

                sh """
                    docker build -t ${IMAGE_NAME} .
                    docker images
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME} > report.txt"
            }
        }

        stage('ECR Push') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds-2', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    // script {
                    //     env.ECR_IMAGE = "${ECR_REPO_URI}:${BUILD_NUMBER}"
                    // }
                    sh """
                        set -e
                        echo 'Checking AWS account'
                        aws sts get-caller-identity
                        echo 'ECR Login'
                        aws ecr get-login-password \
                            --region ap-south-1 | \
                        docker login \
                            --username AWS \
                            --password-stdin \
                            ${ECR_REPO_URI}
                        echo 'Docker Tag'
                        docker tag ${IMAGE_NAME} ${ECR_IMAGE}
                        echo 'Docker Push'
                        docker push ${ECR_IMAGE}
                    """
                }
            }
        }


        stage('Deploy to EKS') {
            steps {
                // Replace with your Jenkins AWS credential ID
                withCredentials([aws(credentialsId: 'aws-creds-2', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh """
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        sed -i 's|_IMAGE_|${ECR_IMAGE}|g' pod.yaml
                        kubectl get nodes
                        kubectl apply -f pod.yaml
                        kubectl get pods
                        kubectl get svc
                    """
                }
            }
        }


    }
}