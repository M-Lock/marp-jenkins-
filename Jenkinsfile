pipeline {
    agent none
    options{
        timeout(time: 20, unit: 'MINUTES') //
        disableConcurrentBuilds() //Evitamos que puedan ocurrir varias builds simultaneamente
        buildDiscarder(logRotator(numToKeepStr: '5')) //Mantenemos solo 5 logs de pipelines anteriores
    }

    triggers{
        pollSCM('H/2 * * * * ') //Ejecutamos cada 2 min
    }

    parameters{
        string (name: 'PUSH_BRANCH', defaultValue: 'main', description: 'rama a la que pushear el fichero')
        string(name: 'EMAIL' , defaultValue: 'mjimsan2001@gmail.com' , description: 'email para realizar el push')
    }

    stages {
        stage ('Build') {
            agent {
                dockerfile true
            }
            steps{
               sh 'which marp' //Comprobamos la instalación de marp
            }
        }

        stage ('Create pdf') {
            agent {
                dockerfile true //reutilizamos la imagen, que ya tiene instalada la dependencia de marp
            }
            steps{
                sh 'marp diapositivas.md -o diapositivas.${BUILD_NUMBER}.pdf' //Ejecuto marp en el workspace
            }
            post{
                success{ //necesitamos saber cuando queremos hacer el post
                    archiveArtifacts artifacts: 'diapositivas.*.pdf', fingerprint: true //guardamos el artefacto. El asterisco ahí indica cualquier pdf que contenga diapositivas en el nombre
                }
            }
            
        }
        //Este Stage podría estar unida a la anterior sin problema, la hemos dividido por visibilidad
        stage ('Push the file') {
            agent {
                dockerfile true
            }
            steps{
                withCredentials([usernamePassword(credentialsId: 'github_credentials', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]){
                    sh """ 
                        git checkout ${params.PUSH_BRANCH}
                        git config user.name ${GH_USER}
                        git config user.email ${params.EMAIL}
                        git add diapositivas.*.pdf
                        git commit -m "generado pdf con las diapositivas numero ${BUILD_NUMBER}"
                        git push https://${GH_USER}:${GH_TOKEN}@github.com/M-Lock/marp-jenkins-.git ${params.PUSH_BRANCH}
                    """
                }
            }
        }
    }
}