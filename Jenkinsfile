pipeline {
    agent none
    options{
        timeout(time: 20, unit: 'MINUTES') //
        disableConcurrentBuilds() //Evitamos que puedan ocurrir varias builds simultaneamente
        buildDiscarter(logRotator(numToKeepStr: '5')) //Mantenemos solo 5 logs de pipelines anteriores
    }

    triggers{
        pollSCM('H/2 * * * * ') //Ejecutamos cada 2 min. mejor que el webhook para ir probando
    }

    parameters{
        string (name: 'PUSH_BRANCH', defaultValue: 'main' description: 'rama a la que pushear el fichero')
    }

    stages {
        stage ('Build') {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    dir '/P3-Jenkins'
                }
            }
            steps{
                sh 'npm install -g marp-cli' //Instalamos la dependencia de marp; Podría ir sin problemas en el dockerfile
            }
        }

        stage ('Create pdf') {
            agent {label 'master'} //Si no comparten los dos stages agent
            steps{
                sh 'git clone https://github.com/uca-iiss/MJ_MM-practicas-DEVOPS' //Clono el repositorio para poder acceder al md
                sh 'cd MJ_MM-practicas-DEVOPS/P3-Jenkins && marp diapositivas.md -o diapostivas.${BUILD_NUMBER}.pdf' //Me muevo a la carpeta y ejecuto marp. Esto no puede ir separado
                //Porque cada sh se realiza en un terminal distinto
            }
            post{
                archiveArtifacts artifacts: 'MJ-MM-practicas-DEVOPS/P3-Jenkins/diapostivas.${BUILD_NUMBER}.pdf', fingerprint: true //guardamos el artefacto
            }
            
        }
        /* 
        stage ('Push the file') {
            agent {label 'master'} //Puede darse el caso de que se ejecuten en máquinas distintas y se rompa el pipeline
            steps{
                withCredentials([usernamePassword(credentialsId: 'github_credentials', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]){
                    sh '''
                        git config user.name ${GH_USER}
                        git config user.email "mjimsan2001@gmail.com"
                        cd MJ_MM-practicas-DEVOPS/P3-Jenkins
                        git add diapostivas.${BUILD_NUMBER}.pdf
                        git commit -m "generado pdf con las diapositivas numero ${BUILD_NUMBER}"
                        git push https://${GH_USER}:${GH_TOKEN}@github.com/uca-iiss/MJ_MM-practicas-DEVOPS ${params.PUSH_BRANCH}
                    '''
                }
            }
        }*/
    }
}