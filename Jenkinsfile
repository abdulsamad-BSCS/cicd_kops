pipeline {

    agent { label 'KOPS' }

    environment {
        registry           = "ihsan0000/vproappdock"
        registryCredential = "dockerhub"      // Jenkins credential ID (Docker Hub user + access token)
        NAMESPACE          = "prod"
    }

    stages {

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build("${registry}:V${BUILD_NUMBER}")
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V${BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused docker image') {
            steps {
                sh "docker rmi ${registry}:V${BUILD_NUMBER} || true"
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'mysonarscanner4'   // must match Manage Jenkins > Tools name
            }
            steps {
                withSonarQubeEnv('sonar-pro') {        // must match Manage Jenkins > System > SonarQube servers name
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=. '''
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Kubernetes Deploy to prod') {
            steps {
                // create the prod namespace if it does not exist
                sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"

                // deploy the helm chart into prod
                sh """helm upgrade --install --force vprofile-stack helm/vprofilecharts \
                      --namespace ${NAMESPACE} \
                      --set appimage=${registry}:V${BUILD_NUMBER}"""

                // check that the pods are coming up
                sh "kubectl get pods -n ${NAMESPACE}"
            }
        }
    }
}
