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
        stage('Skip generated commit') {
            agent any
            steps {
                script {
                    def lastMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
                    def changedFiles = sh(returnStdout: true, script: 'git diff-tree --no-commit-id --name-only -r HEAD').trim()
                        .split('\n')
                        .findAll { it }
                    def onlyGeneratedPdfs = changedFiles && changedFiles.every { it ==~ /diapositivas\.\d+\.pdf/ }

                    env.SKIP_GENERATED_COMMIT = (lastMessage.startsWith('generado pdf con las diapositivas numero') && onlyGeneratedPdfs).toString()

                    if (env.SKIP_GENERATED_COMMIT == 'true') {
                        currentBuild.description = 'Saltada por commit autogenerado de PDF'
                        echo 'El ultimo commit solo contiene PDFs generados por Jenkins. Se saltan los stages restantes para evitar un bucle.'
                    }
                }
            }
        }

        stage('Build') {
            when {
                beforeAgent true
                expression { env.SKIP_GENERATED_COMMIT != 'true' }
            }
            agent {
                dockerfile true
            }
            steps {
                sh 'which marp' // Comprobamos la instalacion de marp
            }
        }

        stage('Create pdf') {
            when {
                beforeAgent true
                expression { env.SKIP_GENERATED_COMMIT != 'true' }
            }
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
            when {
                beforeAgent true
                expression { env.SKIP_GENERATED_COMMIT != 'true' }
            }
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
            when {
                beforeAgent true
                expression { env.SKIP_GENERATED_COMMIT != 'true' }
            }
            agent any
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true // Detenemos el pipeline si SonarQube no supera el quality gate
                }
            }
        }

        stage('Push the file') {
            when {
                beforeAgent true
                expression { env.SKIP_GENERATED_COMMIT != 'true' }
            }
            agent {
                dockerfile true
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github_credentials', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        git checkout "$PUSH_BRANCH"
                        git fetch origin "$PUSH_BRANCH"
                        git reset --hard "origin/$PUSH_BRANCH"
                        git config user.name "$GH_USER"
                        git config user.email "$EMAIL"
                        git add "diapositivas.${BUILD_NUMBER}.pdf"
                        if ! git diff --cached --quiet; then
                            git commit -m "generado pdf con las diapositivas numero ${BUILD_NUMBER}"
                            git push "https://${GH_USER}:${GH_TOKEN}@github.com/M-Lock/marp-jenkins-.git" "$PUSH_BRANCH"
                        else
                            echo "No hay cambios que subir para esta build."
                        fi
                    ''' // Jenkins ya mantiene en su volumen el repo copiado
                }
            }
        }
    }
}
