pipeline {
    agent any

    environment {
        variavelExemplo = 'exemplo.com'
        //## ESSAS VARIÁVEIS TAMBÉM PODEM SER CONFIGURADAS GLOBALMENTE, PARA UTILIZA-LAS LOCALMENTE, BASTA DESCOMENTAR E CONFIGURAR ####

    // urlPortainer  = 'https://127.0.0.1:9443'
    // userPortainer = 'USER_PORTAINER'
    // passwordPortainer = 'PASS_PORTAINER'
    // SwarmID = 'PORTAINER-SWARMID'
    // endpointIdPortainer = '1'
    }

    stages {
                stage('Preparação e Autenticação') {
                    steps {
                        sh 'apt-get update && apt-get install -y jq'

                script {
                            def urlPortainer = env.urlPortainer
                            def userPortainer = env.userPortainer
                            def passwordPortainer = env.passwordPortainer

                            def jwtToken = sh(script:  """
                                curl --request POST -k --url $urlPortainer/api/auth \
                                --header 'Content-Type: application/json' \
                                --data '{ "password": "$passwordPortainer", "username": "$userPortainer" }' | jq -r '.jwt'
                            """, returnStdout: true).trim()

                            // Obtém o valor do token do JSON
                            def jwt = jwtToken

                            echo "Valor do JWT: $jwt"

                            if (jwt) {
                        env.authPortainer = jwt
                            } else {
                        error 'API Authentication Failed'
                            }

                            def gitUrl = sh(returnStdout: true, script: 'git config --get remote.origin.url').trim()
                            echo "Git Repository URL: $gitUrl"
                            // Extrair o nome do repositório do URL
                            def gitRepoName = gitUrl.tokenize('/')[-1].replaceFirst(/\.git$/, '')

                            // Armazenar o nome do repositório em uma variável de ambiente
                            env.gitRepoName = gitRepoName
                }
                    }
                }

                stage('Atualização da Imagem Docker') {
                    steps {
                        script {
                            def jwt = env.authPortainer
                            def urlPortainer = env.urlPortainer
                            def endpointIdPortainer = env.endpointIdPortainer

                            print('Lendo arquivo para ver se existe.')
                            readFile(file: 'Dockerfile', encoding: 'UTF-8')

                            def tagImage = gitRepoName + ':latest'
                            print('preparando para subir - ' + tagImage)

                            def encodedTagImage = URLEncoder.encode(tagImage, 'UTF-8')

                            // sh """curl --request POST -k \
                            //   --url '$urlPortainer/api/endpoints/$endpointIdPortainer/docker/build?dockerfile=Dockerfile&t=$encodedTagImage' \
                            //   --header 'Accept: application/json, text/plain, */*' \
                            //   --header 'Authorization: Bearer $jwt' \
                            //   --header 'Content-Type: multipart/form-data' \
                            //   --form dockerfile=@./Dockerfile
                            //   """
                            // sleep 5

                            //curl --request POST --url 'https://192.168.7.215:9443/api/publicImage' | jq -r '.[] | select(.errorDetail != null).errorDetail.code'

                           def erroString = sh(script: """
                              curl --request POST -k \
                              --url '$urlPortainer/api/endpoints/$endpointIdPortainer/docker/build?dockerfile=Dockerfile&t=$encodedTagImage' \
                              --header 'Accept: application/json, text/plain, */*' \
                              --header 'Authorization: Bearer $jwt' \
                              --header 'Content-Type: multipart/form-data' \
                              --form dockerfile=@./Dockerfile
                            """, returnStdout: true).trim()

                            def pattern = /"errorDetail":\s*\{/
                            def hasErrorDetail = erroString =~ pattern

                            print(erroString)
                            print("verificando se foi erro "+hasErrorDetail)
                             if (hasErrorDetail.find()) {
                                 print("Falha no build")
                                error 'image não buildada no portainer.'
                            } else {
                                echo "No 'errorDetail' found in the JSON string."
                            }

                        //     sh """
                        //    curl --request GET \
                        //     --url 'https://192.168.7.215:9443/api/endpoints/4/docker/images/json?all=0' \
                        //     --header 'Accept: application/json, text/plain, */*' \
                        //     --header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MiwidXNlcm5hbWUiOiJtam9yZGFuIiwicm9sZSI6MSwic2NvcGUiOiJkZWZhdWx0IiwiZm9yY2VDaGFuZ2VQYXNzd29yZCI6ZmFsc2UsImV4cCI6MTY5OTIxMzUyMSwiaWF0IjoxNjk5MTg0NzIxfQ.AtYnKi6IpI96X1QpSOXCJq-kBPyQF6AJDkU6NdqDsRs' \
                        //     --header 'Cache-Control: no-cache' | jq -r '.[] | select(.RepoTags | contains(["service-registration-check:latest"])).Id'

                        //      """
                        //
                        }
                    }
                }

                stage('Derrubada de Serviços Existentes') {
            steps {
                script {
                            String jwt = env.authPortainer
                            String urlPortainer = env.urlPortainer
                            String endpointIdPortainer = env.endpointIdPortainer

                            String gitRepoName = env.gitRepoName
                            print(gitRepoName)

                            String idStack = sh(script: """
                            curl --request GET -k \
                            --url '$urlPortainer/api/stacks'\
                            --header 'Accept: application/json, text/plain, */*'\
                            --header 'Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7'\
                            --header 'Authorization: Bearer $jwt'\
                            --header 'Cache-Control: no-cache' | jq -r '.[] | select(.Name == "$gitRepoName").Id'

                            """, returnStdout: true).trim()

                            print('ID Stack resultante: ' + idStack)

                            if (idStack) {
                        print('Deletando stack ' + idStack)

                        sh """
                                    curl --request DELETE -k \
                                    --url '$urlPortainer/api/stacks/$idStack?endpointId=$endpointIdPortainer&external=false' \
                                    --header 'Accept: application/json, text/plain, */*' \
                                    --header 'Authorization: Bearer $jwt'
                                    """

                        sleep 30
                            }else {
                        print('Stack não existe, não foi preciso excluir.')
                            }
                }
            }
                }

                stage('Subida do Serviço Atualizador') {
                    steps {
                        script {
                            String jwt = env.authPortainer
                            String urlPortainer = env.urlPortainer
                            String swarmID = env.SwarmID

                            String endpointIdPortainer = env.endpointIdPortainer

                            String gitRepoName = env.gitRepoName
                            print(gitRepoName)

                            sh """
                             curl --request POST -k \
                            --url '$urlPortainer/api/stacks/create/swarm/file?endpointId=$endpointIdPortainer' \
                            --header 'Accept: application/json, text/plain, */*' \
                            --header 'Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7' \
                            --header 'Authorization: Bearer $jwt' \
                            --header 'content-type: multipart/form-data' \
                            --form Name=$gitRepoName \
                            --form file=@./docker-compose.yml \
                            --form 'Env=[]' \
                            --form Webhook= \
                            --form SwarmID=$swarmID
                                                """

                    def idNStack = sh(script: """
                            curl --request GET -k \
                            --url '$urlPortainer/api/stacks'\
                            --header 'Accept: application/json, text/plain, */*'\
                            --header 'Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7'\
                            --header 'Authorization: Bearer $jwt'\
                            --header 'Cache-Control: no-cache' | jq -r '.[] | select(.Name == "$gitRepoName").Id'

                            """, returnStdout: true).trim()
                            if (idNStack) {
                        print('Copilado com sucesso!')
                            }else {
                        error 'Stack não encontrada no portainer.'
                            }
                        }
                    }
                }

                stage('Verificação de Serviço Implementado') {
                    steps {
                        script {
                            String jwt = env.authPortainer
                            String urlPortainer = env.urlPortainer
                            String gitRepoName = env.gitRepoName

                            String idNStack = sh(script: """
                            curl --request GET -k \
                            --url '$urlPortainer/api/stacks'\
                            --header 'Accept: application/json, text/plain, */*'\
                            --header 'Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7'\
                            --header 'Authorization: Bearer $jwt'\
                            --header 'Cache-Control: no-cache' | jq -r '.[] | select(.Name == "$gitRepoName").Id'

                            """, returnStdout: true).trim()
                            print('Novo ID Stack publicada' + idNStack)

                            if (idNStack) {
                        print('Copilado com sucesso!')
                            }else {
                        error 'Stack não encontrada no portainer.'
                            }
                        }
                    }
                }
    }
}
