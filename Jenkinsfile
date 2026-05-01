pipeline {
    agent any

    environment {
        GEMINI_API_KEY   = credentials('gemini-api-key')
        COMPOSE_FILE     = 'docker-compose.yml'
        APP_NETWORK      = 'voting-app-chaos-demo_app-net'
        CHAOS_IMAGE      = 'chaos-controller:latest'
        CHAOS_CONTAINER  = 'chaos-controller-ci'
    }

    stages {

        stage('Build & Start Voting App') {
            steps {
                echo '🔨 Building and starting the Voting App stack...'
                sh "docker-compose -f ${COMPOSE_FILE} up --build -d"
                echo '✅ Voting App is building...'
            }
        }

        stage('Start ChaosController') {
            steps {
                echo '🔥 Starting ChaosController...'
                sh "docker rm -f ${CHAOS_CONTAINER} 2>/dev/null || true"
                sh """
                    docker run -d \
                        --name ${CHAOS_CONTAINER} \
                        --network ${APP_NETWORK} \
                        -p 5050:5050 \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -e GEMINI_API_KEY=${GEMINI_API_KEY} \
                        ${CHAOS_IMAGE}
                """
                sh '''
                    for i in $(seq 1 30); do
                        curl -sf http://localhost:5050/status > /dev/null && echo "ChaosController ready!" && exit 0
                        echo "Waiting for ChaosController... ($i/30)"
                        sleep 5
                    done
                    echo "ChaosController failed to start!" && exit 1
                '''
                echo '✅ ChaosController is ready.'
            }
        }

        stage('Application Health Check') {
            steps {
                echo '🏥 Verifying all services are healthy...'
                sh '''
                    for i in $(seq 1 20); do
                        curl -sf http://localhost:8082 > /dev/null && echo "Vote app healthy!" && exit 0
                        echo "Waiting for Vote app... ($i/20)"
                        sleep 5
                    done
                    echo "Vote app failed to become healthy!" && exit 1
                '''
                echo '✅ Voting App is healthy and ready for chaos testing.'
            }
        }

        stage('Resilience Testing Gate') {
            steps {
                script {
                    def agentHost = sh(script: "hostname -I | awk '{print \$1}'", returnStdout: true).trim()

                    echo """
╔══════════════════════════════════════════════════════════════╗
║           🔥  RESILIENCE TESTING GATE  🔥                    ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║   Dashboard: http://${agentHost}:5050                        ║
║                                                              ║
║   Your Voting App services have been auto-discovered.        ║
║   Open the dashboard to:                                     ║
║     • Inject faults (latency, crash, CPU, packet loss...)   ║
║     • Run 3-phase chaos experiments                          ║
║     • Get AI-powered remediation reports (Gemini)            ║
║                                                              ║
║   When satisfied, come back here and click APPROVE.          ║
║   Click REJECT to fail the build.                            ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                    """

                    input(
                        id: 'ResilienceApproval',
                        message: 'Resilience Testing Complete?',
                        ok: 'Approve — System is resilient, continue to deploy'
                    )

                    echo '✅ Resilience gate APPROVED.'
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                echo '🚀 Resilience gate passed — deploying to production...'
                echo '(In a real scenario, this would run: kubectl apply, helm upgrade, etc.)'
                echo '✅ Deployed successfully!'
            }
        }
    }

    post {
        always {
            echo '🧹 Cleaning up...'
            sh "docker stop ${CHAOS_CONTAINER} 2>/dev/null || true"
            sh "docker rm ${CHAOS_CONTAINER} 2>/dev/null || true"
            sh "docker-compose -f ${COMPOSE_FILE} down 2>/dev/null || true"
        }
        failure {
            echo '❌ Pipeline failed or was rejected.'
        }
        success {
            echo '✅ Pipeline succeeded!'
        }
    }
}