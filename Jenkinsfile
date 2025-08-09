pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCESS_KEY_ID     = credentials('jenkins-aws-secret-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
        ECR_BASE = '<your-aws-account-id>.dkr.ecr.${AWS_REGION}.amazonaws.com'
        KUBE_CONFIG = credentials('kubeconfig-jenkins') // Jenkins credential for kubeconfig
        SONARQUBE_URL = 'http://<sonarqube-host>:9000' // SonarQube server URL
        SONARQUBE_TOKEN = credentials('sonarqube-token') // Jenkins secret text credential for SonarQube token
        NEXUS_URL = 'http://<nexus-host>:8081' // Nexus server URL
        NEXUS_CREDENTIALS = credentials('nexus-jenkins') // Jenkins username/password or token credential for Nexus
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONARQUBE_SCANNER_HOME = tool 'SonarQubeScanner'
                SONAR_HOST_URL = env.SONARQUBE_URL
                SONAR_TOKEN = env.SONARQUBE_TOKEN
            }
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    parallel (
                        'Cart': { dir('src/cart') { sh "${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner" } },
                        'Catalog': { dir('src/catalog') { sh "${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner" } },
                        'Orders': { dir('src/orders') { sh "${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner" } },
                        'Checkout': { dir('src/checkout') { sh "${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner" } },
                        'UI': { dir('src/ui') { sh "${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner" } }
                    )
                }
            }
        }

        stage('Build & Test Services') {
            parallel {
                stage('Cart') {
                    steps {
                        dir('src/cart') {
                            sh './mvnw clean package'
                            sh './mvnw test'
                        }
                    }
                }
                stage('Catalog') {
                    steps {
                        dir('src/catalog') {
                            sh 'go test ./...'
                        }
                    }
                }
                stage('Orders') {
                    steps {
                        dir('src/orders') {
                            sh './mvnw clean package'
                            sh './mvnw test'
                        }
                    }
                }
                stage('Checkout') {
                    steps {
                        dir('src/checkout') {
                            sh 'npm install'
                            sh 'npm test'
                        }
                    }
                }
                stage('UI') {
                    steps {
                        dir('src/ui') {
                            sh './mvnw clean package'
                            sh './mvnw test'
                        }
                    }
                }
            }
        }

        stage('Publish Artifacts to Nexus') {
            steps {
                parallel (
                    'Cart': {
                        dir('src/cart') {
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                nexusUrl: env.NEXUS_URL,
                                groupId: 'com.example',
                                version: "${env.GIT_COMMIT}",
                                repository: 'maven-releases',
                                credentialsId: env.NEXUS_CREDENTIALS,
                                artifacts: [
                                    [artifactId: 'cart', classifier: '', file: 'target/carts-0.0.1-SNAPSHOT.jar', type: 'jar']
                                ]
                            )
                        }
                    },
                    'Catalog': {
                        dir('src/catalog') {
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                nexusUrl: env.NEXUS_URL,
                                groupId: 'com.example',
                                version: "${env.GIT_COMMIT}",
                                repository: 'maven-releases',
                                credentialsId: env.NEXUS_CREDENTIALS,
                                artifacts: [
                                    [artifactId: 'catalog', classifier: '', file: 'main', type: 'go']
                                ]
                            )
                        }
                    },
                    'Orders': {
                        dir('src/orders') {
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                nexusUrl: env.NEXUS_URL,
                                groupId: 'com.example',
                                version: "${env.GIT_COMMIT}",
                                repository: 'maven-releases',
                                credentialsId: env.NEXUS_CREDENTIALS,
                                artifacts: [
                                    [artifactId: 'orders', classifier: '', file: 'target/orders-0.0.1-SNAPSHOT.jar', type: 'jar']
                                ]
                            )
                        }
                    },
                    'Checkout': {
                        dir('src/checkout') {
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                nexusUrl: env.NEXUS_URL,
                                groupId: 'com.example',
                                version: "${env.GIT_COMMIT}",
                                repository: 'npm-releases',
                                credentialsId: env.NEXUS_CREDENTIALS,
                                artifacts: [
                                    [artifactId: 'checkout', classifier: '', file: 'dist', type: 'node']
                                ]
                            )
                        }
                    },
                    'UI': {
                        dir('src/ui') {
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                nexusUrl: env.NEXUS_URL,
                                groupId: 'com.example',
                                version: "${env.GIT_COMMIT}",
                                repository: 'maven-releases',
                                credentialsId: env.NEXUS_CREDENTIALS,
                                artifacts: [
                                    [artifactId: 'ui', classifier: '', file: 'target/ui-0.0.1-SNAPSHOT.jar', type: 'jar']
                                ]
                            )
                        }
                    }
                )
            }
        }

        stage('Build & Push Docker Images') {
            parallel {
                stage('Cart') {
                    steps {
                        script {
                            def tag = "${env.GIT_COMMIT}"
                            def repo = "${ECR_BASE}/cart"
                            docker.build("${repo}:${tag}", "src/cart").push()
                        }
                    }
                }
                stage('Catalog') {
                    steps {
                        script {
                            def tag = "${env.GIT_COMMIT}"
                            def repo = "${ECR_BASE}/catalog"
                            docker.build("${repo}:${tag}", "src/catalog").push()
                        }
                    }
                }
                stage('Orders') {
                    steps {
                        script {
                            def tag = "${env.GIT_COMMIT}"
                            def repo = "${ECR_BASE}/orders"
                            docker.build("${repo}:${tag}", "src/orders").push()
                        }
                    }
                }
                stage('Checkout') {
                    steps {
                        script {
                            def tag = "${env.GIT_COMMIT}"
                            def repo = "${ECR_BASE}/checkout"
                            docker.build("${repo}:${tag}", "src/checkout").push()
                        }
                    }
                }
                stage('UI') {
                    steps {
                        script {
                            def tag = "${env.GIT_COMMIT}"
                            def repo = "${ECR_BASE}/ui"
                            docker.build("${repo}:${tag}", "src/ui").push()
                        }
                    }
                }
            }
        }

        stage('Update Helm Charts') {
            steps {
                script {
                    def tag = "${env.GIT_COMMIT}"
                    sh "sed -i 's|tag:.*|tag: \"${tag}\"|g' src/cart/chart/values.yaml"
                    sh "sed -i 's|tag:.*|tag: \"${tag}\"|g' src/catalog/chart/values.yaml"
                    sh "sed -i 's|tag:.*|tag: \"${tag}\"|g' src/orders/chart/values.yaml"
                    sh "sed -i 's|tag:.*|tag: \"${tag}\"|g' src/checkout/chart/values.yaml"
                    sh "sed -i 's|tag:.*|tag: \"${tag}\"|g' src/ui/chart/values.yaml"
                    sh "git config user.email 'jenkins@yourdomain.com'"
                    sh "git config user.name 'Jenkins'"
                    sh "git add src/*/chart/values.yaml"
                    sh "git commit -m 'ci: update image tags for all services to ${tag}' || true"
                    sh "git push origin HEAD:main"
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-jenkins', variable: 'KUBECONFIG')]) {
                    sh "helm upgrade --install cart src/cart/chart --namespace retail-store --kubeconfig $KUBECONFIG"
                    sh "helm upgrade --install catalog src/catalog/chart --namespace retail-store --kubeconfig $KUBECONFIG"
                    sh "helm upgrade --install orders src/orders/chart --namespace retail-store --kubeconfig $KUBECONFIG"
                    sh "helm upgrade --install checkout src/checkout/chart --namespace retail-store --kubeconfig $KUBECONFIG"
                    sh "helm upgrade --install ui src/ui/chart --namespace retail-store --kubeconfig $KUBECONFIG"
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
