pipeline {
// este jenkisfile será utilizado para el reto numero cinco, por lo que he tenido que modificar el nombre 
    agent any

    environment {
        bucketName = "todo-list-aws-bucket-${UUID.randomUUID().toString()}"
    }

    stages {
        stage('Get Code') {
            steps {
                echo 'Clonando el repositorio...'
                git branch: 'develop', url: 'https://github.com/CarballoFrancisco/todo-list-aws.git'
            }
        }

        stage('Static Test') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        echo 'Running Static Analysis with Flake8 and Bandit...'
                        sh 'flake8 --exit-zero --format=pylint src >flake8.out'
                        sh 'cat flake8.out'
                        recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]

                        sh 'python3 -m venv venv'
                        sh './venv/bin/bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}]: {msg}"'
                        sh 'cat bandit.out'
                        recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
                    }
                }
            }
        }

        stage('SAM') {
            steps {
                echo 'Construyendo, validando y desplegando la aplicación SAM...'
                sh 'sam build'
                sh 'sam validate'

                sh 'aws cloudformation delete-stack --stack-name todo-list-stack --region us-east-1'
                sh 'aws cloudformation wait stack-delete-complete --stack-name todo-list-stack --region us-east-1'
                sh "aws s3api create-bucket --bucket ${bucketName} --region us-east-1"

                sh """
                    sam deploy --no-confirm-changeset \
                    --stack-name todo-list-stack \
                    --capabilities CAPABILITY_IAM \
                    --region us-east-1 \
                    --s3-bucket ${bucketName} \
                    --no-fail-on-empty-changeset \
                    --parameter-overrides Stage=staging
                """
            }
        }

       stage('REST API Integration Tests') {
    steps {
        script {
            echo 'Running REST API integration tests using curl...'

            def baseUrl = sh(script: 'aws cloudformation describe-stacks --stack-name todo-list-stack --region us-east-1 --query "Stacks[0].Outputs[?OutputKey==\'BaseUrlApi\'].OutputValue" --output text', returnStdout: true).trim()
            def apiUrl = "${baseUrl}/todos"

            // GET todos
            def responseCode = sh(script: "curl -s -o /dev/null -w '%{http_code}' -X GET ${apiUrl}", returnStdout: true).trim()
            echo "GET /todos - Código de estado: ${responseCode} (200 significa éxito)"
            if (responseCode != '200') {
                error "Error al acceder a la API. Código de estado: ${responseCode}"
            }

            // POST todos
            def postData = '{"text": "Test task", "userId": "12345"}'
            def postResponse = sh(script: """
                curl -s -X POST ${apiUrl} \
                -H "Content-Type: application/json" \
                -d '${postData}' \
                -w "%{http_code}" -o post_response.json
            """, returnStdout: true).trim()

            echo "POST /todos - Código de estado: ${postResponse} (200 significa éxito)"
            if (postResponse != '200') {
                error "POST request failed! Expected 200, got ${postResponse}"
            }

            // Extraer el ID del TODO creado
            def todoId = sh(script: "jq -r '.body | fromjson | .id' post_response.json", returnStdout: true).trim()
            echo "Nuevo TODO creado con ID: ${todoId}"

            // GET todos/{id}
            def getTodoResponse = sh(script: "curl -s -o /dev/null -w '%{http_code}' -X GET ${apiUrl}/${todoId}", returnStdout: true).trim()
            echo "GET /todos/${todoId} - Código de estado: ${getTodoResponse} (200 significa éxito)"
            if (getTodoResponse != '200') {
                error "Error al obtener el TODO con ID ${todoId}. Código de estado: ${getTodoResponse}"
            }

            // PUT todos/{id} (actualizar antes de eliminar)
            def putData = '{"text": "Updated task", "userId": "12345", "checked": true}'
            def putResponse = sh(script: """
                curl -s -X PUT ${apiUrl}/${todoId} \
                -H "Content-Type: application/json" \
                -d '${putData}' \
                -w "%{http_code}" -o put_response.json
            """, returnStdout: true).trim()

            echo "PUT /todos/${todoId} - Código de estado: ${putResponse} (200 significa éxito)"
            if (putResponse != '200') {
                error "PUT request failed! Expected 200, got ${putResponse}"
            }

            // DELETE todos/{id}
            def deleteResponseCode = sh(script: "curl -s -o /dev/null -w '%{http_code}' -X DELETE ${apiUrl}/${todoId}", returnStdout: true).trim()
            echo "DELETE /todos/${todoId} - Código de estado: ${deleteResponseCode} (200 significa éxito)"
            if (deleteResponseCode != '200') {
                error "Error al eliminar el TODO. Código de estado: ${deleteResponseCode}"
            }
        }
    }
}


        stage('Promote to Master') {
            steps {
                script {
                    echo 'Promoting the code to master...'
                    withCredentials([usernamePassword(credentialsId: 'GitHub-Token-Jenkins', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git config --global credential.helper store
                            git config --global user.name "$GITHUB_USER"
                            git config --global user.email "$GITHUB_USER@users.noreply.github.com"

                            git checkout master
                            git pull origin master
                            git merge develop --no-ff -m "Promoting version to production"
                            git push https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/CarballoFrancisco/todo-list-aws.git master
                        '''
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completada con éxito!'
        }
        failure {
            echo 'La ejecución de la pipeline ha fallado.'
        }
    }
}
