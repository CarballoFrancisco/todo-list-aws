pipeline {
    // Este Jenkinsfile es para realizar el caso práctico cinco, donde el pipeline realiza el despliegue en producción una vez que el código está en la rama 'master'
    // Por lo que para que jenkins reconozca el archivo de la rama he tenido que cambiarle el nombre

    agent any

    environment {
        bucketName = "todo-list-aws-bucket-${UUID.randomUUID().toString()}"
    }

    stages {
        stage('Get Code') {
            steps {
                echo 'Clonando el repositorio desde master...'
                git branch: 'master', url: 'https://github.com/CarballoFrancisco/todo-list-aws.git'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Desplegando la aplicación en producción...'
                sh 'sam build'
                sh 'sam validate'

                sh 'aws cloudformation delete-stack --stack-name todo-list-prod-stack --region us-east-1'
                sh 'aws cloudformation wait stack-delete-complete --stack-name todo-list-prod-stack --region us-east-1'
                sh "aws s3api create-bucket --bucket ${bucketName} --region us-east-1"

                sh """
                    sam deploy --no-confirm-changeset \
                    --stack-name todo-list-prod-stack \
                    --capabilities CAPABILITY_IAM \
                    --region us-east-1 \
                    --s3-bucket ${bucketName} \
                    --no-fail-on-empty-changeset \
                    --parameter-overrides Stage=production
                """
            }
        }

        stage('REST API Read-Only Tests') {
            steps {
                script {
                    echo 'Running REST API integration tests using curl...'

                    def baseUrl = sh(script: 'aws cloudformation describe-stacks --stack-name todo-list-stack --region us-east-1 --query "Stacks[0].Outputs[?OutputKey==\'BaseUrlApi\'].OutputValue" --output text', returnStdout: true).trim()
                    def apiUrl = "${baseUrl}/todos"

                    // GET todos
                    def responseCode = sh(script: "curl -s -o /dev/null -w '%{http_code}' -X GET ${apiUrl}", returnStdout: true).trim()
                    
                    // Comentario explicando que el código 200 significa éxito
                    echo "Código de estado recibido: ${responseCode} (200 significa que GET se ejecutó correctamente)"

                    if (responseCode == '200') {
                        echo "La API responde correctamente."
                    } else {
                        error "Error al acceder a la API. Código de estado: ${responseCode}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Despliegue en producción completado con éxito!'
        }
        failure {
            echo 'Fallo en la ejecución del pipeline de producción.'
        }
    }
}
