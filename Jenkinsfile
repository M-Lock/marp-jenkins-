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
        string(name: 'MARKDOWN_FILE', defaultValue: 'diapositivas.md', description: 'archivo markdown a procesar')
    }

    stages {
        // Stages posibles para este pipeline:
        // - Checkout desde SCM
        // - Instalacion de dependencias
        // - Generacion del PDF
        // - Archivado del artefacto
        // - Analisis de SonarQube
        // - Quality Gate
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
                script {
                    env.SLIDES_BASENAME = params.MARKDOWN_FILE.replaceFirst(/\.md$/, '')
                }
                sh 'marp "$MARKDOWN_FILE" -o "${SLIDES_BASENAME}.${BUILD_NUMBER}.pdf"' // Ejecutamos marp en el workspace
            }
            post {
                success {
                    archiveArtifacts artifacts: "${env.SLIDES_BASENAME}.*.pdf", fingerprint: true // Guardamos el artefacto generado
                }
            }
        }

        stage('SonarQube Analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest' // Usamos una imagen que ya incluye sonar-scanner
                    args '--network host' // Permitimos que el contenedor temporal alcance SonarQube
                    reuseNode true
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonar_token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=marp-jenkins \
                              -Dsonar.projectName=marp-jenkins \
                              -Dsonar.sources=. \
                              -Dsonar.exclusions=**/*.pdf,**/.git/**,**/.scannerwork/** \
                              -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
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
    }
}
