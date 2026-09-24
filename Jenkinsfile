pipeline {
    agent { label 'KOPS' }

    environment {
        registry           = "ihsan0000/vproappdock"
        registryCredential = "dockerhub"
        NAMESPACE          = "prod"
    }

    stages {

        stage('Compile') {
            steps {
                script {
                    if (fileExists('pom.xml')) {
                        sh 'mvn -B clean compile -DskipTests'
                        env.JAVA_BINARIES = 'target/classes'
                    } else {
                        env.JAVA_BINARIES = '.'
                    }
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            environment { scannerHome = tool 'mysonarscanner4' }
            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=${JAVA_BINARIES} '''
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script { dockerImage = docker.build("${registry}:V${BUILD_NUMBER}") }
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
                sh "docker image prune -f || true"
            }
        }

        stage('Kubernetes Deploy to prod') {
            steps {
                sh "kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"
                sh """helm upgrade --install --force vprofile-stack helm/vprofilecharts \
                      --namespace ${NAMESPACE} \
                      --set appimage=${registry}:V${BUILD_NUMBER}"""
                sh "kubectl get pods -n ${NAMESPACE}"
            }
        }
    }
}
