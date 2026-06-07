pipeline {
    agent any
    options {
        timeout(time: 120, unit: 'MINUTES')
        timestamps()
    }

    environment {
        REPO_URL       = "https://github.com/WebGoat/WebGoat.git"
        RECIPIENT_MAIL = "fatoundour@esp.sn"
        REPORTS_DIR    = "${WORKSPACE}\\reports"
        DC_HOME        = "C:\\dependency-check"
        WEBGOAT_PORT   = "8888"
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout WebGoat') {
            steps {
                git url: "${REPO_URL}", branch: 'main'
                bat "if not exist reports mkdir reports"
                echo "Code WebGoat clone"
            }
        }

        stage('SAST - Bearer') {
            steps {
                script {
                    echo "=== SAST : Analyse Bearer ==="
                    bat """
                        docker run --rm ^
                            -v "%WORKSPACE%:/src" ^
                            bearer/bearer:latest ^
                            scan /src ^
                            --format json ^
                            --output /src/reports/bearer-report.json ^
                            --exit-code 0 ^
                            --scanner sast,secrets ^
                            --quiet
                    """
                    echo "SAST Bearer termine - rapport : reports/bearer-report.json"
                }
            }
        }

        stage('SCA - Dependency-Check') {
            steps {
                script {
                    echo "=== SCA : OWASP Dependency-Check ==="
                    bat """
                        "${DC_HOME}\\bin\\dependency-check.bat" ^
                            --project "WebGoat-SCA" ^
                            --scan "%WORKSPACE%" ^
                            --format HTML ^
                            --format JSON ^
                            --out "%WORKSPACE%\\reports" ^
                            --failOnCVSS 11 ^
                            --enableExperimental ^
                            --disableOssIndex
                    """
                }
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing         : true,
                        alwaysLinkToLastBuild: true,
                        keepAll              : true,
                        reportDir            : 'reports',
                        reportFiles          : 'dependency-check-report.html',
                        reportName           : 'SCA - Dependency-Check'
                    ])
                }
            }
        }

        stage('Deploy WebGoat pour DAST') {
            steps {
                script {
                    echo "=== Deploiement WebGoat pour ZAP ==="
                    bat "docker stop webgoat 2>nul & exit 0"
                    bat "docker rm   webgoat 2>nul & exit 0"
                    bat """
                        docker run -d ^
                            --name webgoat ^
                            -p 8888:8080 ^
                            -p 9191:9090 ^
                            webgoat/webgoat:latest
                    """
                    sleep(time: 45, unit: 'SECONDS')
                    echo "WebGoat disponible sur http://localhost:${WEBGOAT_PORT}/WebGoat"
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    echo "=== DAST : OWASP ZAP Scan ==="
                    // exit 0 absorbe le code 2 retourne par ZAP quand il detecte des warnings
                    // Un vrai FAIL (FAIL-NEW > 0) reste visible dans le rapport HTML
                    bat """
                        docker run --rm ^
                            --network host ^
                            -v "%WORKSPACE%\\reports:/zap/wrk" ^
                            ghcr.io/zaproxy/zaproxy:stable ^
                            zap-baseline.py ^
