// Ejecuta el comando con sh (Linux/Mac) o bat (Windows) según el agente
def ejecutar(String comando) {
    if (isUnix()) {
        sh comando
    } else {
        bat comando
    }
}

pipeline {
    agent any

    tools {
        // Nombres definidos en: Administrar Jenkins > Tools
        maven 'Maven3'
        jdk 'JDK17'
    }

    environment {
        APP_PORT      = '8085'
        APP_JAR       = 'target/psw-pipeline-base-0.0.1-SNAPSHOT.jar'
        JMETER_HOME   = 'C:\\apache-jmeter-5.6.3\\apache-jmeter-5.6.3'   // Ajustar a la ruta de tu JMeter
        JMETER_USERS  = '50'                        // Usuarios concurrentes (50 - 100)
        SLACK_CHANNEL = '#jenkins'                  // Canal de Slack
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                ejecutar 'mvn -B clean package'
            }
        }

        stage('Análisis SonarQube') {
            steps {
                // 'SonarQube' = nombre del servidor en: Administrar Jenkins > System > SonarQube servers
                withSonarQubeEnv('SonarQube') {
                    ejecutar 'mvn -B sonar:sonar -Dsonar.projectKey=psw-pipeline-base -Dsonar.projectName=psw-pipeline-base'
                }
            }
        }

        stage('Pruebas JMeter') {
            steps {
                script {
                    if (isUnix()) {
                        sh """
                            nohup java -jar ${APP_JAR} > app.log 2>&1 &
                            echo \$! > app.pid
                            for i in \$(seq 1 30); do
                                curl -s http://localhost:${APP_PORT}/actuator/health && break
                                sleep 2
                            done
                            ${JMETER_HOME}/bin/jmeter -n -f -t jmeter/carga-psw.jmx -Jusuarios=${JMETER_USERS} -l jmeter/resultados.jtl -e -o jmeter/reporte
                            kill \$(cat app.pid) || true
                        """
                    } else {
                        bat """
                            start "psw-app" /B java -jar ${APP_JAR} > app.log 2>&1
                            powershell -NoProfile -Command "for(\$i=0;\$i -lt 30;\$i++){try{Invoke-WebRequest http://localhost:${APP_PORT}/actuator/health -UseBasicParsing | Out-Null; exit 0}catch{Start-Sleep 2}}; exit 1"
                            call "%JMETER_HOME%\\bin\\jmeter.bat" -n -f -t jmeter\\carga-psw.jmx -Jusuarios=${JMETER_USERS} -l jmeter\\resultados.jtl -e -o jmeter\\reporte
                            set JMETER_RC=%ERRORLEVEL%
                            for /f "tokens=5" %%a in ('netstat -ano ^| findstr :${APP_PORT} ^| findstr LISTENING') do taskkill /F /PID %%a >nul 2>&1
                            exit /b %JMETER_RC%
                        """
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'jmeter/resultados.jtl, jmeter/reporte/**, app.log', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: "✅ ÉXITO: ${env.JOB_NAME} #${env.BUILD_NUMBER} finalizó correctamente.\n${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: "❌ ERROR: ${env.JOB_NAME} #${env.BUILD_NUMBER} falló.\n${env.BUILD_URL}console"
            )
        }
    }
}
