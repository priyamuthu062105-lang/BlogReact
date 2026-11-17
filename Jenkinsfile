
pipeline {
    agent any

    environment {
        REGISTRY_PREFIX = 'blogreact'
        K3D_CLUSTER = 'habit-cluster'
        KUBE_CONFIG = '/var/lib/jenkins/.kube/config'
        NAMESPACE = 'blogapp'
    }

    stages {

        stage('Backend Tests and Lint') {
            steps {
                dir('backend') {
                    sh '''
                        docker run --rm \
                          -v "$PWD":/app -w /app \
                          node:22-alpine sh -c \
                          "npm ci && npm test -- --runInBand && npm run lint"
                    '''
                }
            }
        }

        stage('Frontend Lint and Build') {
            steps {
                dir('frontend') {
                    sh '''
                        docker run --rm \
                          -v "$PWD":/app -w /app \
                          node:22-alpine sh -c \
                          "npm ci && npm run lint && npm run build"
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    env.IMAGE_TAG = "build-${BUILD_NUMBER}"
                }
                sh '''
                    docker build -t ${REGISTRY_PREFIX}-backend:${IMAGE_TAG} ./backend
                    docker build -t ${REGISTRY_PREFIX}-frontend:${IMAGE_TAG} ./frontend
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy image --scanners vuln --timeout 10m \
                      --exit-code 1 --severity CRITICAL \
                      ${REGISTRY_PREFIX}-backend:${IMAGE_TAG}

                    trivy image --scanners vuln --timeout 10m \
                      --exit-code 1 --severity CRITICAL \
                      ${REGISTRY_PREFIX}-frontend:${IMAGE_TAG}
                '''
            }
        }

        stage('Import Images into k3d') {
            steps {
                sh '''
                    k3d image import \
                      ${REGISTRY_PREFIX}-backend:${IMAGE_TAG} \
                      ${REGISTRY_PREFIX}-frontend:${IMAGE_TAG} \
                      -c ${K3D_CLUSTER}
                '''
            }
        }

        stage('Deploy BlogReact') {
            steps {
                sh '''
                    kubectl --kubeconfig=${KUBE_CONFIG} \
                      set image deployment/backend \
                      backend=${REGISTRY_PREFIX}-backend:${IMAGE_TAG} \
                      -n ${NAMESPACE}

                    kubectl --kubeconfig=${KUBE_CONFIG} \
                      rollout status deployment/backend \
                      -n ${NAMESPACE} --timeout=180s

                    kubectl --kubeconfig=${KUBE_CONFIG} \
                      set image deployment/frontend \
                      frontend=${REGISTRY_PREFIX}-frontend:${IMAGE_TAG} \
                      -n ${NAMESPACE}

                    kubectl --kubeconfig=${KUBE_CONFIG} \
                      rollout status deployment/frontend \
                      -n ${NAMESPACE} --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl --kubeconfig=${KUBE_CONFIG} \
                      get deployments,pods -n ${NAMESPACE}

                    kubectl --kubeconfig=${KUBE_CONFIG} \
                      get services -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo 'BlogReact CI/CD pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs before retrying.'
        }
    }
}

