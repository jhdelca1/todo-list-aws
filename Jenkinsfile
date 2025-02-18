pipeline {
    agent any
   
    stages {
        stage('Get Code') {
            steps {
                // Borra todo lo que haya en el workspace, incluyendo .git
                //deleteDir()
                git branch: 'develop', url: 'https://github.com/jhdelca1/todo-list-aws.git'
                echo "El workspace actual es: ${env.WORKSPACE}"
            }
        }
        stage('Static Test') {
            steps {
                script {
                    echo "Ejecutando análisis estático con Flake8 y Bandit..."
                    // Ejecutar Flake8 y guardar el informe
                    sh 'flake8 --exit-zero --format=pylint src >flake8.out'
                    // Ejecutar Bandit y guardar el informe
                    sh 'bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"|| true'
                    echo "Análisis estático completado. Publicando informes..."
                    recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]
                    recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
                }
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
                    echo "Verificando si el stack está en estado ROLLBACK_COMPLETE..."
                    
                    // Verificar si el stack está en estado ROLLBACK_COMPLETE
                    def stackStatus = sh(
                        script: """
                            aws cloudformation describe-stacks --stack-name todo-list-aws-staging --region us-east-1 \
                                --query 'Stacks[0].StackStatus' --output text || echo 'STACK_NOT_FOUND'
                        """,
                        returnStdout: true
                    ).trim()
                    
                    if (stackStatus == "ROLLBACK_COMPLETE") {
                        echo "Stack en estado ROLLBACK_COMPLETE. Eliminando antes de desplegar..."
                        sh "aws cloudformation delete-stack --stack-name todo-list-aws-staging --region us-east-1"
                        sh "aws cloudformation wait stack-delete-complete --stack-name todo-list-aws-staging --region us-east-1"
                    } else {
                        echo "Stack en estado válido: ${stackStatus}. Continuando con el despliegue..."
                    }
            
            // Despliegue usando la configuración 'staging' de samconfig.toml
                    sh '''
                        sam deploy \
                            --config-env staging \
                            --parameter-overrides "CreateDynamoTable=false" \
                            --no-confirm-changeset \
                            --no-fail-on-empty-changeset
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
                            aws cloudformation describe-stacks --stack-name todo-list-aws-staging --region us-east-1 \
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
                        echo "Ejecutando pruebas REST con Pytest..."
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh '''
                                python3 -m pytest --version
                                python3 -m pytest -s -v --junitxml=result-integration.xml test/integration | tee result-integration.log
                            '''
                        }
                        echo "Analizando el log de pruebas..."
                        def passedTests = sh(
                            script: "grep -c 'PASSED' result-integration.log || echo 0",
                            returnStdout: true
                        ).trim()

                        if (passedTests != "5") {
                            error "No todas las pruebas pasaron correctamente. Se esperaban 5 PASSED, pero se encontraron: ${passedTests}"
                        }

                        echo "Todas las pruebas pasaron correctamente (${passedTests}/5)."
                        echo "Procesando resultados de pruebas..."
                        junit 'result-integration.xml'
                    }
                }
            }
        }
       
        stage('Promote') {
            steps {
                script {
                    echo "Iniciando la promoción de la versión a producción..."
                    // Obtener el token desde los secretos de Jenkins
                    withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'TOKEN')]) {
                        // Configurar usuario de Git (necesario para commits automáticos en Jenkins)
                        sh '''
                            git config --global user.email "jherna21@gmail.com"
                            git config --global user.name "jhdelca1"
                        '''
                        // Cambiar la URL del repositorio para usar el token
                        sh '''
                            git remote set-url origin https://$TOKEN@github.com/jhdelca1/todo-list-aws.git
                        '''
                        // Verificar que estamos en la rama `develop`
                        def currentBranch = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                        if (currentBranch != "develop") {
                            error "La rama actual es '${currentBranch}', pero se esperaba 'develop'. Verifica la configuración del pipeline."
                        }
        
                        echo "Cambio a la rama 'master'..."
                        sh '''
                            git fetch origin
                            git checkout main
                            git pull --rebase origin main
                        '''
                        
    
                        echo "Haciendo merge de 'develop' en 'master'..."

                        sh '''
                            git merge --no-ff develop -m "🔀 Merge automático desde develop a main [Jenkins CI]" || true
                            git checkout --ours CHANGELOG.md  # Mantener versión de develop
                            git add CHANGELOG.md
                            git commit -m "✔ Resolviendo conflicto en CHANGELOG.md manteniendo versión de develop" || true
                        '''
    
                        echo "Subiendo los cambios a GitHub..."
                        sh '''
                            git push origin main
                        '''
    
                        echo "Promoción completada: la versión está lista para master."
                    }
                }
            }
        } 
    }
}
