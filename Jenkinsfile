pipeline {
    agent any
   
    stages {
        stage('Get Code') {
            steps {
                
                git branch: 'main', url: 'https://github.com/jhdelca1/todo-list-aws.git'
                echo "El workspace actual es: ${env.WORKSPACE}"
            }
        }
        stage('Deploy') {
            steps {
                script {
                    echo "Iniciando despliegue con AWS SAM..."
                    // Construcción del código
                    sh 'sam build'
                    // Validación de la plantilla SAM
                    sh 'sam validate --region us-east-1'
                    // Verificar si el stack está en estado ROLLBACK_COMPLETE
                    def stackStatus = sh(
                        script: """
                            aws cloudformation describe-stacks --stack-name todo-list-aws-production --region us-east-1 \
                                --query 'Stacks[0].StackStatus' --output text || echo 'STACK_NOT_FOUND'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    if (stackStatus == "ROLLBACK_COMPLETE") {
                        echo "Stack en estado ROLLBACK_COMPLETE. Eliminando antes de desplegar..."
                        sh "aws cloudformation delete-stack --stack-name todo-list-aws-production --region us-east-1"
                        sh "aws cloudformation wait stack-delete-complete --stack-name todo-list-aws-production --region us-east-1"
                    } else {
                        echo "Stack en estado válido: ${stackStatus}. Continuando con el despliegue..."
                    }
                    // Despliegue usando la configuración 'staging' de samconfig.toml
                    sh '''
                        sam deploy \
                            --config-env production  \
                            --no-confirm-changeset --no-fail-on-empty-changeset
                    '''
                    echo "Despliegue completado exitosamente."
                }
            }
        }    
        
        stage('Rest Test'){
            steps{
                script {
                    echo "Obteniendo la URL del API desde CloudFormation..."

                    // Obtener `BASE_URL` desde los outputs de CloudFormation
                    def BASE_URL = sh(
                        script: """
                            aws cloudformation describe-stacks --stack-name todo-list-aws-production --region us-east-1 \
                                --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --output text
                            """,
                            returnStdout: true
                    ).trim()

                    // Si `BASE_URL` no se obtiene, se detiene el pipeline
                    if (BASE_URL == "" || BASE_URL == "None") {
                        error "No se pudo obtener la URL del API. Verifica que el stack se haya desplegado correctamente."
                    }

                    echo "API URL obtenida: ${BASE_URL}"

                    // Definir `BASE_URL` en el entorno del pipeline
                    withEnv(["BASE_URL=${BASE_URL}"]) {
                        echo "Ejecutando pruebas REST de solo lectura con Pytest..."
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh '''
                                python3 -m pytest --version
                                python3 -m pytest -s -v -k "listtodos or gettodo" --junitxml=result-integration.xml test/integration | tee result-integration.log
                            '''
                        }
    
                        echo "📄 Analizando el log de pruebas..."
                        def passedTests = sh(
                            script: "grep -c 'PASSED' result-integration.log || echo 0",
                            returnStdout: true
                        ).trim()

                        if (passedTests != "2") {
                            error "No todas las pruebas de lectura pasaron correctamente. Se esperaban 2 PASSED, pero se encontraron: ${passedTests}"
                        }

                        echo "Todas las pruebas de lectura pasaron correctamente (${passedTests}/2)."
                        echo "Procesando resultados de pruebas..."
                        junit 'result-integration.xml'
                    }
                }
            }
        }
    }
}
