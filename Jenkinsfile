pipeline {
    agent any
    
    environment {
        repositoryGit = 'https://github.com/alcifmais/alcifroot'
        gitCredentialsId = 'devopsalcif'
    }

    stages {
                stage('Prepare') {
                    steps {
                        sh 'apt-get update && apt-get install -y jq'
                        
                         script {
                            def urlPortainer = env.urlPortainer
                            def userPortainer = env.userPortainer
                            def passwordPortainer = env.passwordPortainer
                            def SwarmID = env.SwarmID
                            def endpointIdPortainer = env.endpointIdPortainer
                            
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
                            
                            
                            
                        }
                        
                    }
                }

                stage('Checkout from GitHub') {
                    steps {
                        script {
                            def gitCredentialsId = env.gitCredentialsId
                            def gitRepoUrl = env.repositoryGit
                                // Extrair o nome do repositório do URL
                                def gitRepoName = gitRepoUrl.tokenize('/')[-1].replaceFirst(/\.git$/, '')

                                // Armazenar o nome do repositório em uma variável de ambiente
                                env.gitRepoName = gitRepoName

                                git branch: 'main', credentialsId: gitCredentialsId, url: gitRepoUrl
                        }
                    }
                }


                stage('Deploy on Portainer') {
                    steps {
                        script {
                            def jwt = env.authPortainer
                            def urlPortainer = env.urlPortainer
                            def SwarmID = env.SwarmID
                            def endpointIdPortainer = env.endpointIdPortainer
                            def dockerComposeFile = sh(script: "find . -name 'docker-compose.yml' | head -n 1", returnStdout: true).trim()
                            print(dockerComposeFile)
                            def gitRepoName = env.gitRepoName
                            print(gitRepoName)
                            
                            def fileContent = readFile(file: 'Dockerfile', encoding: 'UTF-8')
                            def tagImage = gitRepoName + ":latest"
                            def encodedTagImage = URLEncoder.encode(tagImage, "UTF-8")
                            
                            def sanitizedFileContent = fileContent.replaceAll('\\n', '\\n')
                            
                            sh """curl --request POST -k \
                              --url '$urlPortainer/api/endpoints/$endpointIdPortainer/docker/build?dockerfile=Dockerfile&t=$encodedTagImage' \
                              --header 'Accept: application/json, text/plain, */*' \
                              --header 'Authorization: Bearer $jwt' \
                              --header 'Content-Type: multipart/form-data' \
                              --form dockerfile=@./Dockerfile
                              """
                            
                            def idStack = sh(script: """ 
                            curl --request GET -k \
                            --url '$urlPortainer/api/stacks'\
                            --header 'Accept: application/json, text/plain, */*'\
                            --header 'Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7'\
                            --header 'Authorization: Bearer $jwt'\
                            --header 'Cache-Control: no-cache' | jq -r '.[] | select(.Name == "$gitRepoName").Id'

                            """, returnStdout: true).trim()
                            print(idStack)
                            
                            if(idStack){
                                print("Deletando..")
                                
                                  sh """
                                    curl --request DELETE -k \
                                      --url '$urlPortainer/api/stacks/$idStack?endpointId=$endpointIdPortainer&external=false' \
                                      --header 'Accept: application/json, text/plain, */*' \
                                      --header 'Authorization: Bearer $jwt'
                                    """
                                    
                                    sleep 30
                            }
                            
                            
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
                            --form SwarmID=$SwarmID
                                                """
                         def idNStack = sh(script: """ 
                            curl --request GET -k \
                            --url '$urlPortainer/api/stacks'\
                            --header 'Accept: application/json, text/plain, */*'\
                            --header 'Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7'\
                            --header 'Authorization: Bearer $jwt'\
                            --header 'Cache-Control: no-cache' | jq -r '.[] | select(.Name == "$gitRepoName").Id'

                            """, returnStdout: true).trim()      
                            if(idNStack){
                                print("Copilado com sucesso!")
                            }else{
                                   error 'Stack não encontrada no portainer.'
                            }
                            
                        }
                    }
                }
    }
}
