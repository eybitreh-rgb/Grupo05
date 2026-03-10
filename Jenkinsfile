pipeline {
    agent any

    environment {
        REPORT_NAME = 'semgrep-report.json'
        IMAGE_NAME  = 'lab4-app'
        IMAGE_TAG   = 'latest'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/michix14/lab4.git',
                    credentialsId: 'spring-app'
                echo "Código descargado correctamente"
            }
        }

        stage('SAST - Remote Semgrep') {
            steps {
                withCredentials([
                    string(credentialsId: 'server-ssh-user', variable: 'REMOTE_USER'),
                    string(credentialsId: 'server-ip-address', variable: 'REMOTE_IP')
                ]) {
                    sshagent(['ubuntu-server']) {
                        sh '''
                            echo "🧹 Preparando directorio remoto..."

                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                rm -rf /tmp/app-scan &&
                                mkdir -p /tmp/app-scan
                            "

                            echo "📤 Enviando código..."
                            scp -r -o StrictHostKeyChecking=no \
                                $(ls -A | grep -v .git) \
                                ${REMOTE_USER}@${REMOTE_IP}:/tmp/app-scan

                            echo "🚀 Ejecutando Semgrep..."
                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                semgrep --config=auto \
                                --json \
                                --output /tmp/semgrep-report.json \
                                /tmp/app-scan || true
                            "

                            echo "🐳 Construyendo imagen Docker..."
                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                cd /tmp/app-scan &&
                                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                            "

                            echo "🛡️ Analizando contenedor con Snyk..."
                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                snyk container test ${IMAGE_NAME}:${IMAGE_TAG} \
                                --json > /tmp/snyk-report.json || true
                            "

                            echo "🚀 Levantando contenedor para DAST..."
                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                docker run -d --name zap-target \
                                -p 8080:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                            "

                            echo "⏳ Esperando que la app inicie..."
                            sleep 20

                            echo "🛡 Ejecutando DAST con OWASP ZAP..."
                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                docker run --rm \
                                --network=host \
                                -v /tmp:/zap/wrk:rw \
                                zaproxy/zap-stable \
                                zap-baseline.py \
                                -t http://localhost:8080 \
                                -J zap-report.json || true
                            "

                            echo "🧹 Deteniendo contenedor..."
                            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_IP} "
                                docker stop zap-target &&
                                docker rm zap-target
                            "

                            echo "📥 Descargando reportes..."
                            scp -o StrictHostKeyChecking=no \
                                ${REMOTE_USER}@${REMOTE_IP}:/tmp/semgrep-report.json \
                                semgrep-report.json

                            scp -o StrictHostKeyChecking=no \
                                ${REMOTE_USER}@${REMOTE_IP}:/tmp/snyk-report.json \
                                snyk-report.json

                            scp -o StrictHostKeyChecking=no \
                                ${REMOTE_USER}@${REMOTE_IP}:/tmp/zap-report.json \
                                zap-report.json
                        '''
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.json', fingerprint: true
        }

        failure {
            echo "❌ Pipeline falló"
        }

        success {
            echo "✅ Análisis y build completados correctamente"
        }
    }
}
