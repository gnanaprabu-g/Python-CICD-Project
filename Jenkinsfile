// Build the FastAPI image, scan it with Trivy, push to ECR, and deploy to EKS.
//
// Agent (Linux) needs: Docker, AWS CLI v2, kubectl, and bash.
// Jenkins credential: Username with password.
//   Username = AWS access key id
//   Password = AWS secret access key
//   ID       = aws-credentials  (or change AWS_CREDENTIALS_ID below)
//
// EKS node role needs AmazonEC2ContainerRegistryReadOnly so nodes can pull the image.
// The Jenkins IAM user/role needs ECR push, eks:DescribeCluster, and Kubernetes RBAC to apply this manifest.
// Fill in AWS_ACCOUNT_ID and EKS_CLUSTER before the first run.

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        AWS_REGION          = 'ap-south-1'
        AWS_ACCOUNT_ID      = 'YOUR_AWS_ACCOUNT_ID'
        ECR_REPO            = 'fastapi-app'
        EKS_CLUSTER         = 'YOUR_EKS_CLUSTER'
        K8S_NAMESPACE       = 'default'
        AWS_CREDENTIALS_ID  = 'aws-credentials'
        IMAGE_TAG           = "${BUILD_NUMBER}"
        // Fail the build when Trivy finds a fixable HIGH or CRITICAL issue.
        // Set TRIVY_EXIT_CODE to 0 to report only.
        TRIVY_SEVERITY      = 'HIGH,CRITICAL'
        TRIVY_EXIT_CODE     = '1'
        TRIVY_IMAGE         = 'aquasec/trivy:latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker image') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    IMAGE="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
                    docker build -t "${IMAGE}" -t "${ECR_REPO}:${IMAGE_TAG}" .
                    docker image inspect "${IMAGE}" --format 'Built {{.Id}}'
                '''
            }
        }

        stage('Trivy scan') {
            steps {
                sh '''#!/usr/bin/env bash
                    set -euo pipefail
                    IMAGE="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"

                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v "${WORKSPACE}:/work" \
                        -v trivy-cache:/root/.cache \
                        "${TRIVY_IMAGE}" image \
                        --severity "${TRIVY_SEVERITY}" \
                        --ignore-unfixed \
                        --scanners vuln \
                        --format json \
                        --output /work/trivy-report.json \
                        "${IMAGE}"

                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v trivy-cache:/root/.cache \
                        "${TRIVY_IMAGE}" image \
                        --exit-code "${TRIVY_EXIT_CODE}" \
                        --severity "${TRIVY_SEVERITY}" \
                        --ignore-unfixed \
                        --scanners vuln \
                        "${IMAGE}"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${AWS_CREDENTIALS_ID}",
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        IMAGE="${REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"

                        aws ecr describe-repositories \
                            --repository-names "${ECR_REPO}" \
                            --region "${AWS_REGION}" \
                            >/dev/null 2>&1 \
                        || aws ecr create-repository \
                            --repository-name "${ECR_REPO}" \
                            --region "${AWS_REGION}" \
                            --image-scanning-configuration scanOnPush=true

                        aws ecr get-login-password --region "${AWS_REGION}" \
                            | docker login --username AWS --password-stdin "${REGISTRY}"

                        docker push "${IMAGE}"
                        docker tag "${IMAGE}" "${REGISTRY}/${ECR_REPO}:latest"
                        docker push "${REGISTRY}/${ECR_REPO}:latest"
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${AWS_CREDENTIALS_ID}",
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''#!/usr/bin/env bash
                        set -euo pipefail
                        IMAGE="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"

                        aws eks update-kubeconfig \
                            --region "${AWS_REGION}" \
                            --name "${EKS_CLUSTER}"

                        kubectl get namespace "${K8S_NAMESPACE}" >/dev/null 2>&1 \
                            || kubectl create namespace "${K8S_NAMESPACE}"

                        sed "s|_IMAGE_|${IMAGE}|g" pod.yaml \
                            | kubectl apply -n "${K8S_NAMESPACE}" -f -

                        kubectl rollout status deployment/loadbalancer-pod \
                            -n "${K8S_NAMESPACE}" \
                            --timeout=300s

                        kubectl get deploy,svc -l app=hello -n "${K8S_NAMESPACE}"
                        kubectl get svc loadbalancer-svc \
                            -n "${K8S_NAMESPACE}" \
                            -o jsonpath='Load balancer: {.status.loadBalancer.ingress[0].hostname}{"\\n"}' \
                            || true
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${ECR_REPO}:${IMAGE_TAG} to EKS cluster ${EKS_CLUSTER}."
        }
        failure {
            echo 'Pipeline failed. Trivy report (if the scan ran) is archived as trivy-report.json.'
        }
    }
}
