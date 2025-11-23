pipeline {
    agent any
    
    tools{ maven 'maven3'}
      parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment{SCANNER_HOME= tool 'sonar-scanner'
        IMAGE_NAME= "narupraveen/bank"
        TAG="${params.DOCKER_TAG}" //comes from parameter
        KUBE_NAMESPACE= 'webapps'
    }

    stages {
        stage('git-checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/NaruPraveenKumar/blue-green-deployment-.git'
            }
        }
        stage('compile') {
            steps {
                sh 'mvn compile'
            }
        }
        stage('testcases') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }
        stage('fs scan') {
            steps {
                sh "trivy fs  --format table -o fs.html ."
            }
        }
        stage('sonarqube-analysis') {
            steps {
                withSonarQubeEnv('sonar') {
           sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=multitier -Dsonar.projectName=multitier -Dsonar.java.binaries=target"
                                           }
            }
        }
        stage('qualitychecks')
        {
        steps{
        timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
                                        }
            }
        }
        stage('build') {
            steps {
             sh 'mvn package -DskipTests=True'   
            }
        }
          stage('publichartifatcs') {
            steps {
              withMaven(globalMavenSettingsConfig: 'maven-settings', maven: 'maven3', traceability: true) {
              //sh 'mvn deploy -DskipTests=True'
                                                                        }
            }
        }
         stage('build and tag image') {
            steps {
             script{
                 // This step should not normally be used in your script. Consult the inline help for details.
withDockerRegistry(credentialsId: 'docker-creds') {
               sh "docker build -t ${IMAGE_NAME}:${TAG} ."
    // some block
                                                      }
             }
            }
         }
                stage('image scan') {
            steps {
                sh "trivy image  --format table -o fs.html  ${IMAGE_NAME}:${TAG} "
            }
        } 
             stage('build and push image') {
            steps {
             script{
                 // This step should not normally be used in your script. Consult the inline help for details.
withDockerRegistry(credentialsId: 'docker-creds') {
               sh "docker push  ${IMAGE_NAME}:${TAG}"
    // some block
                                                      }
             }
            }
         }
          stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3ABF5D64A431B981722D6FD9ABC28C83.yl4.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"  // Ensure you have the MySQL deployment YAML ready
                    }
                }
            }
        }   
        
         stage('Deploy SVC-APP') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3ABF5D64A431B981722D6FD9ABC28C83.yl4.ap-south-1.eks.amazonaws.com') {
                        sh """ if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                              fi
                        """
                   }
                }
            }
        }
            stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = ""
                    if (params.DEPLOY_ENV == 'blue') {
                        deploymentFile = 'app-deployment-blue.yml'
                    } else {
                        deploymentFile = 'app-deployment-green.yml'
                    }

                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3ABF5D64A431B981722D6FD9ABC28C83.yl4.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        } 
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV

                    // Always switch traffic based on DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3ABF5D64A431B981722D6FD9ABC28C83.yl4.ap-south-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${KUBE_NAMESPACE}
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://3ABF5D64A431B981722D6FD9ABC28C83.yl4.ap-south-1.eks.amazonaws.com') {
                        sh """
                        kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                        kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
        }
    }
