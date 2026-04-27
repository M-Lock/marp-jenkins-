pipeline {
    agent none

    options {
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds() // Evitamos que puedan ocurrir varias builds simultaneamente
        buildDiscarder(logRotator(numToKeepStr: '5')) // Mantenemos solo 5 logs de pipelines anteriores
    }

    triggers {
        pollSCM('H/2 * * * *') // Ejecutamos cada 2 min
    }

    parameters {
        string(name: 'PUSH_BRANCH', defaultValue: 'main', description: 'rama a la que pushear el fichero')
        string(name: 'EMAIL', defaultValue: 'mjimsan2001@gmail.com', description: 'email para realizar el push')
    }

    stages {
        stage('Build') {
            agent {
                dockerfile true
            }
            steps {
                sh 'which marp' // Comprobamos la instalacion de marp
            }
        }

        stage('Create pdf') {
            agent {
                dockerfile true // Reutilizamos la imagen, que ya tiene instalada la dependencia de marp
            }
            steps {
                sh 'marp diapositivas.md -o diapositivas.${BUILD_NUMBER}.pdf' // Ejecutamos marp en el workspace
            }
            post {
                success {
                    archiveArtifacts artifacts: 'diapositivas.*.pdf', fingerprint: true // Guardamos el artefacto generado
                }
            }
        }

        stage('SonarQube Analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest' // Usamos una imagen que ya incluye sonar-scanner
                    reuseNode true
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        # Enviamos el proyecto actual a SonarQube para su analisis
                        sonar-scanner \
                          -Dsonar.projectKey=marp-jenkins \
                          -Dsonar.projectName=marp-jenkins \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/*.pdf,**/.git/**,**/.scannerwork/**
                    '''
                }
            }
        }

        stage('Quality Gate') {
            agent any
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true // Detenemos el pipeline si SonarQube no supera el quality gate
                }
            }
        }

        stage('Push the file') {
            agent {
                dockerfile true
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github_credentials', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh """
                        git checkout ${params.PUSH_BRANCH}
                        git config user.name ${GH_USER}
                        git config user.email ${params.EMAIL}
                        git add diapositivas.*.pdf
                        git commit -m "generado pdf con las diapositivas numero ${BUILD_NUMBER}"
                        git push https://${GH_USER}:${GH_TOKEN}@github.com/M-Lock/marp-jenkins-.git ${params.PUSH_BRANCH}
                    """ // Jenkins ya mantiene en su volumen el repo copiado
                }
            }
        }
    }
}
