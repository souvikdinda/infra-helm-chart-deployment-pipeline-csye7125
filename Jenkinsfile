def consumerappVersion
pipeline {
    agent any 
    
    environment {
        GCLOUD_PROJECT = 'csye7125-gke-f8c10e6f'
        GCLOUD_CLUSTER = 'my-gke-cluster'
        GCLOUD_REGION = 'us-east1'
        INFRA_NAMESPACE = 'kafka'
    }
    
    stages {
        stage('Authenticating with GKE') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'gke_service_account_key', variable: 'GCLOUD_SERVICE_KEY')]) {
                        sh "gcloud auth activate-service-account --key-file=${GCLOUD_SERVICE_KEY}"
                    }

                    sh "gcloud config set project ${GCLOUD_PROJECT}"
                    sh "gcloud container clusters get-credentials my-gke-cluster --region ${GCLOUD_REGION}"

                    echo "Authenticated with GKE..."

                }
            }
        }

        stage('Checking out the code') {
            steps{
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: 'main']],
                        userRemoteConfigs: [[credentialsId: 'github_token', url: 'https://github.com/csye7125-fall2023-group07/infra-helm-chart.git']]
                    ])
                }
            }
        }

        stage('Updating values.yaml') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github_token', usernameVariable: 'GH_USERNAME', passwordVariable: 'GH_TOKEN')]) {
                        consumerappVersion = sh(
                            script: """
                                git ls-remote --tags https://\${GH_TOKEN}@github.com/csye7125-fall2023-group07/consumer-app.git |
                                awk '{print \$2}' |
                                awk -F"/" '{print \$NF}' |
                                awk -F"v" '{print \$NF}' |
                                sort -V |
                                tail -n 1
                            """,
                            returnStdout: true
                        ).trim()
                        sh "echo 'consumerappVersion: ${consumerappVersion}'"

                      
                    }

                    sh "sed -i 's/tag: [0-9].[0-9].[0-9]/tag: ${consumerappVersion}/' values.yaml"
                    
                }
            }
        }

        stage('Check and Create Namespace') {
            steps {
                script {
                    def namespaceExists = sh(script: "kubectl get namespace ${INFRA_NAMESPACE}", returnStatus: true)
                    
                    if (namespaceExists != 0) {
                        sh "kubectl create namespace ${INFRA_NAMESPACE}"
                        echo "Namespace ${INFRA_NAMESPACE} created."
                    } else {
                        echo "Namespace ${INFRA_NAMESPACE} already exists."
                    }
                }
            }
        }
        
      stage('Label Namespace for Istio Injection') {
            steps {
                script {
                    sh "kubectl label namespace ${INFRA_NAMESPACE} istio-injection=enabled --overwrite"
                    echo "Namespace ${INFRA_NAMESPACE} labeled for Istio injection."
                }
            }
        }

        stage('Deploying to GKE') {
            steps {
                sh 'helm upgrade --install infra -n ${INFRA_NAMESPACE} .'
            }
        }

        stage('Revoke GKE access') {
            steps {
                sh 'gcloud auth revoke --all'
            }
            
        }
    }
}
