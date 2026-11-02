pipeline {
    agent any

    //  environment {
    //     AWS_REGION = 'ap-south-1'
    //     AWS_ACCOUNT_ID = 'YOUR_AWS_ACCOUNT_ID'
    //     ECR_REPO = '978355780378.dkr.ecr.ap-south-1.amazonaws.com'
    //     EKS_CLUSTER = 'python-eks-cluster'
    //     FULL_IMAGE = 'python-app'
    //}

    stages {
        stage('Pull Code From GitHub') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/gnanaprabu-g/Python-CICD-Project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=python-app \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    env.IMAGE_NAME = "python-app:${BUILD_NUMBER}"
                }

                sh '''
                    docker build -t "$IMAGE_NAME" .
                    docker images
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image ${IMAGE_NAME} > report.txt'
            }
        }

        stage('ECR Push') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds-2', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        env.ECR_IMAGE = "978355780378.dkr.ecr.ap-south-1.amazonaws.com/python-app:${BUILD_NUMBER}"
                    }
                    sh '''
                        set -e
                        echo "Checking AWS account"
                        aws sts get-caller-identity
                        echo "ECR Login"
                        aws ecr get-login-password \
                            --region ap-south-1 | \
                        docker login \
                            --username AWS \
                            --password-stdin \
                            978355780378.dkr.ecr.ap-south-1.amazonaws.com
                        echo "Docker Tag"
                        docker tag python-app:${BUILD_NUMBER} "$ECR_IMAGE"
                        echo "Docker Push"
                        docker push "$ECR_IMAGE"
                    '''
                }
            }
        }


        stage('Deploy to EKS') {
            steps {
                // Replace with your Jenkins AWS credential ID
                withCredentials([aws(credentialsId: 'aws-creds-2', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws eks update-kubeconfig --region ap-south-1 --name demo-cluster-1
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        ECR_IMAGE=978355780378.dkr.ecr.ap-south-1.amazonaws.com/python-app:${BUILD_NUMBER}
                        sed -i "s|_IMAGE_|$ECR_IMAGE|g" pod.yaml
                        kubectl get nodes
                        kubectl apply -f pod.yaml
                        kubectl get pods
                        kubectl get svc
                    '''
                }
            }
        }


    }
}