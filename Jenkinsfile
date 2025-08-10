pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        ECR_BASE = '<your-aws-account-id>.dkr.ecr.${AWS_REGION}.amazonaws.com'
        SONARQUBE_SERVER = 'SonarQubeServer' // Jenkins SonarQube server name
        SERVICES = ['cart', 'catalog', 'orders', 'checkout', 'ui']
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    script {
                        parallel SERVICES.collectEntries { svc ->
                            ["${svc}": {
                                dir("src/${svc}") {
                                    sh "${tool 'SonarQubeScanner'}/bin/sonar-scanner -Dsonar.projectKey=${svc} -Dsonar.sources=."
                                }
                            }]
                        }
                    }
                }
            }
        }

        stage('Build & Test Services') {
            steps {
                script {
                    parallel(
                        'Cart': { dir('src/cart') { sh './mvnw clean package && ./mvnw test' } },
                        'Catalog': { dir('src/catalog') { sh 'go test ./...' } },
                        'Orders': { dir('src/orders') { sh './mvnw clean package && ./mvnw test' } },
                        'Checkout': { dir('src/checkout') { sh 'npm install && npm test' } },
                        'UI': { dir('src/ui') { sh './mvnw clean package && ./mvnw test' } }
                    )
                }
            }
        }

        stage('Publish Artifacts to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-jenkins', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    script {
                        parallel(
                            'Cart': {
                                dir('src/cart') {
                                    nexusArtifactUploader(
                                        nexusVersion: 'nexus3',
                                        protocol: 'http',
                                        nexusUrl: 'http://172.31.43.168:8081',
                                        groupId: 'com.example',
                                        version: "${env.GIT_COMMIT}",
                                        repository: 'maven-releases',
                                        credentialsId: 'nexus-jenkins',
                                        artifacts: [[artifactId: 'cart', file: 'target/carts-0.0.1-SNAPSHOT.jar', type: 'jar']]
                                    )
                                }
                            },
                            'Catalog': {
                                dir('src/catalog') {
                                    sh "tar -czvf catalog.tar.gz main"
                                    nexusArtifactUploader(
                                        nexusVersion: 'nexus3',
                                        protocol: 'http',
                                        nexusUrl: 'http://172.31.43.168:8081',
                                        groupId: 'com.example',
                                        version: "${env.GIT_COMMIT}",
                                        repository: 'binary-releases',
                                        credentialsId: 'nexus-jenkins',
                                        artifacts: [[artifactId: 'catalog', file: 'catalog.tar.gz', type: 'tar.gz']]
                                    )
                                }
                            },
                            'Orders': {
                                dir('src/orders') {
                                    nexusArtifactUploader(
                                        nexusVersion: 'nexus3',
                                        protocol: 'http',
                                        nexusUrl: 'http://172.31.43.168:8081',
                                        groupId: 'com.example',
                                        version: "${env.GIT_COMMIT}",
                                        repository: 'maven-releases',
                                        credentialsId: 'nexus-jenkins',
                                        artifacts: [[artifactId: 'orders', file: 'target/orders-0.0.1-SNAPSHOT.jar', type: 'jar']]
                                    )
                                }
                            },
                            'Checkout': {
                                dir('src/checkout') {
                                    nexusArtifactUploader(
                                        nexusVersion: 'nexus3',
                                        protocol: 'http',
                                        nexusUrl: 'http://172.31.43.168:8081',
                                        groupId: 'com.example',
                                        version: "${env.GIT_COMMIT}",
                                        repository: 'npm-releases',
                                        credentialsId: 'nexus-jenkins',
                                        artifacts: [[artifactId: 'checkout', file: 'dist', type: 'node']]
                                    )
                                }
                            },
                            'UI': {
                                dir('src/ui') {
                                    nexusArtifactUploader(
                                        nexusVersion: 'nexus3',
                                        protocol: 'http',
                                        nexusUrl: 'http://172.31.43.168:8081',
                                        groupId: 'com.example',
                                        version: "${env.GIT_COMMIT}",
                                        repository: 'maven-releases',
                                        credentialsId: 'nexus-jenkins',
                                        artifacts: [[artifactId: 'ui', file: 'target/ui-0.0.1-SNAPSHOT.jar', type: 'jar']]
                                    )
                                }
                            }
                        )
                    }
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'jenkins-aws-secret-key',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} \
                        | docker login --username AWS --password-stdin ${ECR_BASE}
                    """
                    script {
                        parallel SERVICES.collectEntries { svc ->
                            ["${svc}": {
                                docker.build("${ECR_BASE}/${svc}:${env.GIT_COMMIT}", "src/${svc}")
                                    .push()
                            }]
                        }
                    }
                }
            }
        }

        stage('Update Helm Charts') {
            steps {
                script {
                    SERVICES.each { svc ->
                        sh "sed -i 's|tag:.*|tag: \"${env.GIT_COMMIT}\"|g' src/${svc}/chart/values.yaml"
                    }
                    sh "git config user.email 'jenkins@yourdomain.com'"
                    sh "git config user.name 'Jenkins'"
                    sh "git add src/*/chart/values.yaml"
                    sh "git commit -m 'ci: update image tags for all services to ${env.GIT_COMMIT}' || true"
                    withCredentials([usernamePassword(credentialsId: 'git-jenkins', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh "git push https://${GIT_USER}:${GIT_PASS}@<your-repo-url>.git HEAD:main"
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-jenkins', variable: 'KUBECONFIG')]) {
                    script {
                        SERVICES.each { svc ->
                            sh "helm upgrade --install ${svc} src/${svc}/chart --namespace retail-store --kubeconfig $KUBECONFIG"
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            script {
                SERVICES.each { svc ->
                    sh "helm rollback ${svc} 1 --namespace retail-store || true"
                }
            }
        }
        always {
            cleanWs()
        }
    }
}
