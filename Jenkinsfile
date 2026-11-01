pipeline {
 
    agent any
 
    environment {

        AWS_REGION = 'ap-south-1'

        EKS_CLUSTER = 'python-eks-cluster'
 
        DOCKER_IMAGE = 'YOUR_DOCKERHUB_USERNAME/python-app'
 
        K8S_NAMESPACE = 'python-app'

        K8S_DEPLOYMENT = 'python-app'

        K8S_CONTAINER = 'python-app'

    }
 
    stages {
 
        stage('Checkout') {

            steps {

                checkout scm

            }

        }

 
        stage('Python Unit Testing') {

            steps {

                sh '''

                    . venv/bin/activate

                    pytest -v

                '''

            }

        }
 
        stage('SonarQube Analysis') {

            steps {

                script {

                    def scannerHome = tool 'SonarScanner'
 
                    withSonarQubeEnv('SonarQube') {

                        sh """

                            ${scannerHome}/bin/sonar-scanner

                        """

                    }

                }

            }

        }
 
        stage('Quality Gate') {

            steps {

                timeout(time: 5, unit: 'MINUTES') {

                    script {

                        def qg = waitForQualityGate()
 
                        if (qg.status != 'OK') {

                            error "SonarQube Quality Gate Failed: ${qg.status}"

                        }

                    }

                }

            }

        }
 
        stage('Docker Build') {

            steps {

                script {

                    env.IMAGE_TAG = "${BUILD_NUMBER}"

                    env.FULL_IMAGE =

                        "${DOCKER_IMAGE}:${IMAGE_TAG}"

                }
 
                sh '''

                    docker build -t $FULL_IMAGE .

                    docker images

                '''

            }

        }
 
        stage('Docker Hub Push') {

            steps {

                withCredentials([

                    usernamePassword(

                        credentialsId: 'docker-hub-credentials',

                        usernameVariable: 'DOCKER_USER',

                        passwordVariable: 'DOCKER_PASS'

                    )

                ]) {

                    sh '''

                        echo "$DOCKER_PASS" | docker login \

                            -u "$DOCKER_USER" --password-stdin
 
                        docker push "$FULL_IMAGE"
 
                        docker logout

                    '''

                }

            }

        }
 
        stage('Configure EKS') {

            steps {

                sh '''

                    aws sts get-caller-identity
 
                    aws eks update-kubeconfig \

                        --region "$AWS_REGION" \

                        --name "$EKS_CLUSTER"
 
                    kubectl get nodes

                '''

            }

        }
 
        stage('Deploy to EKS') {

            steps {

                sh '''

                    kubectl create namespace "$K8S_NAMESPACE" \

                        --dry-run=client -o yaml | kubectl apply -f -
 
                    sed "s|IMAGE_PLACEHOLDER|$FULL_IMAGE|g" \

                        deployment.yaml | kubectl apply -f -
 
                    kubectl apply -f service.yaml
 
                    kubectl rollout status \

                        deployment/"$K8S_DEPLOYMENT" \

                        -n "$K8S_NAMESPACE" \

                        --timeout=300s

                '''

            }

        }
 
        stage('Verify Deployment') {

            steps {

                sh '''

                    echo "Checking Kubernetes Deployment"
 
                    kubectl get deployments \

                        -n "$K8S_NAMESPACE"
 
                    kubectl get pods \

                        -n "$K8S_NAMESPACE" -o wide
 
                    kubectl get services \

                        -n "$K8S_NAMESPACE"

                '''

            }

        }

    }

}