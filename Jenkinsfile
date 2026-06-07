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
                    echo "WebGoat disponible sur http://localhost:8888/WebGoat"
                }
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                script {
                    echo "=== DAST : OWASP ZAP Scan ==="
                    bat """
                        docker run --rm ^
                            --network host ^
                            -v "%WORKSPACE%\\reports:/zap/wrk" ^
                            ghcr.io/zaproxy/zaproxy:stable ^
                            zap-baseline.py ^
                            -t http://localhost:8888/WebGoat ^
                            -r zap-report.html ^
                            -J zap-report.json ^
                            -l WARN ^
                            --auto
                    """
                    echo "DAST ZAP termine - rapport : reports/zap-report.html"
                }
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing         : true,
                        alwaysLinkToLastBuild: true,
                        keepAll              : true,
                        reportDir            : 'reports',
                        reportFiles          : 'zap-report.html',
                        reportName           : 'DAST - ZAP Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            bat "docker stop webgoat 2>nul & exit 0"
            bat "docker rm   webgoat 2>nul & exit 0"

            archiveArtifacts(
                artifacts: 'reports/**/*',
                allowEmptyArchive: true,
                fingerprint: true
            )

            emailext(
                to                 : "${RECIPIENT_MAIL}",
                subject            : "[DevSecOps] WebGoat - Build #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
                mimeType           : 'text/html',
                attachmentsPattern : 'reports/bearer-report.json',
                body               : """
<html>
<body style="font-family:Arial,sans-serif; padding:20px;">
<h2 style="color:#1A3A5C;">Rapport DevSecOps - WebGoat</h2>
<p>Build declenche automatiquement par un <b>push GitHub</b>.</p>
<table border="1" cellpadding="8" style="border-collapse:collapse; width:500px;">
  <tr style="background:#1A3A5C; color:white;">
    <th>Champ</th><th>Valeur</th>
  </tr>
  <tr><td><b>Job</b></td><td>${env.JOB_NAME}</td></tr>
  <tr style="background:#f5f7fa;">
    <td><b>Build N</b></td><td>${env.BUILD_NUMBER}</td>
  </tr>
  <tr>
    <td><b>Statut</b></td>
    <td style="color:${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'};">
      <b>${currentBuild.currentResult}</b>
    </td>
  </tr>
  <tr style="background:#f5f7fa;">
    <td><b>Duree</b></td><td>${currentBuild.durationString}</td>
  </tr>
  <tr>
    <td><b>Rapport SAST</b></td>
    <td><a href="${env.BUILD_URL}artifact/reports/bearer-report.json">Bearer JSON</a></td>
  </tr>
  <tr style="background:#f5f7fa;">
    <td><b>Rapport SCA</b></td>
    <td><a href="${env.BUILD_URL}SCA_20-_20Dependency-Check">Dependency-Check HTML</a></td>
  </tr>
  <tr>
    <td><b>Rapport DAST</b></td>
    <td><a href="${env.BUILD_URL}DAST_20-_20ZAP_20Report">ZAP HTML</a></td>
  </tr>
  <tr style="background:#f5f7fa;">
    <td><b>Jenkins</b></td>
    <td><a href="${env.BUILD_URL}">Ouvrir le build</a></td>
  </tr>
</table>
<br/>
<p style="color:gray; font-size:11px;">
  Outils : Bearer (SAST) + OWASP Dependency-Check (SCA) + OWASP ZAP (DAST)
</p>
</body>
</html>
                """
            )

            echo "=== Rapports envoyes a ${RECIPIENT_MAIL} ==="
        }

        success {
            echo "Pipeline DevSecOps termine avec succes !"
        }

        failure {
            echo "Pipeline en echec - consulter les rapports ci-dessus"
        }
    }
}
