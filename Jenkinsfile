pipeline {

    agent any

    options {
        timeout(time: 120, unit: 'MINUTES')
        timestamps()
    }

    environment {
        REPO_URL       = "https://github.com/VOTRE_USERNAME/WebGoat.git"
        RECIPIENT_MAIL = "fatoundour@esp.sn"
        REPORTS_DIR    = "${WORKSPACE}\\reports"
        DC_HOME        = "C:\\dependency-check"
        WEBGOAT_PORT   = "8080"
    }

    triggers {
        githubPush()
    }

    stages {

        // ── STAGE 1 : Checkout ────────────────────────────────
        stage('Checkout WebGoat') {
            steps {
                git url: "${REPO_URL}", branch: 'main'
                bat "if not exist reports mkdir reports"
                echo "Code WebGoat cloné"
            }
        }

        // ── STAGE 2 : SAST - Bearer ───────────────────────────
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

        // ── STAGE 3 : SCA - Dependency-Check ─────────────────
        stage('SCA - Dependency-Check') {
            steps {
                script {
                    echo "=== SCA : OWASP Dependency-Check ==="
                    // exit code 0 pour ne pas bloquer les stages suivants
                    // on analyse les résultats manuellement dans post
                    bat """
                        "${DC_HOME}\\bin\\dependency-check.bat" ^
                            --project "WebGoat-SCA" ^
                            --scan "%WORKSPACE%" ^
                            --format HTML ^
                            --format JSON ^
                            --out "%WORKSPACE%\\reports" ^
                            --failOnCVSS 10 ^
                            --enableExperimental ^
                            --suppression "%WORKSPACE%\\suppression.xml" ^
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
                        reportDir            : "${WORKSPACE}\\reports",
                        reportFiles          : 'dependency-check-report.html',
                        reportName           : 'SCA - Dependency-Check'
                    ])
                }
            }
        }

        // ── STAGE 4 : Déployer WebGoat pour DAST ─────────────
        stage('Deploy WebGoat pour DAST') {
            steps {
                script {
                    echo "=== Déploiement WebGoat pour ZAP ==="
                    bat "docker stop webgoat 2>nul & exit 0"
                    bat "docker rm   webgoat 2>nul & exit 0"
                    bat """
                        docker run -d ^
                            --name webgoat ^
                            -p ${WEBGOAT_PORT}:8080 ^
                            -p 9090:9090 ^
                            webgoat/goat-and-wolf
                    """
                    // Attendre que WebGoat soit prêt
                    sleep(time: 30, unit: 'SECONDS')
                    echo "WebGoat disponible sur http://localhost:${WEBGOAT_PORT}/WebGoat"
                }
            }
        }

        // ── STAGE 5 : DAST - OWASP ZAP ───────────────────────
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
                            -t http://localhost:${WEBGOAT_PORT}/WebGoat ^
                            -r zap-report.html ^
                            -J zap-report.json ^
                            -x zap-report.xml ^
                            -I ^
                            --hook=/zap/auth_hook.py 2>nul & exit 0
                    """
                    echo "DAST ZAP terminé - rapport : reports/zap-report.html"
                }
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing         : true,
                        alwaysLinkToLastBuild: true,
                        keepAll              : true,
                        reportDir            : "${WORKSPACE}\\reports",
                        reportFiles          : 'zap-report.html',
                        reportName           : 'DAST - ZAP Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            // Stopper WebGoat
            bat "docker stop webgoat 2>nul & exit 0"
            bat "docker rm   webgoat 2>nul & exit 0"

            // Archiver tous les rapports
            archiveArtifacts(
                artifacts: 'reports/**/*',
                allowEmptyArchive: true,
                fingerprint: true
            )

            // Envoi du rapport par mail
            emailext(
                to                 : "${RECIPIENT_MAIL}",
                subject            : "[DevSecOps] WebGoat — Build #${env.BUILD_NUMBER} — ${currentBuild.currentResult}",
                mimeType           : 'text/html',
                attachmentsPattern : 'reports/bearer-report.json',
                body               : """
<html>
<body style="font-family:Arial,sans-serif; padding:20px;">
<h2 style="color:#1A3A5C;">Rapport DevSecOps — WebGoat</h2>
<p>Build déclenché automatiquement par un <b>push GitHub</b>.</p>
<table border="1" cellpadding="8" style="border-collapse:collapse; width:500px;">
  <tr style="background:#1A3A5C; color:white;">
    <th>Champ</th><th>Valeur</th>
  </tr>
  <tr><td><b>Job</b></td><td>${env.JOB_NAME}</td></tr>
  <tr style="background:#f5f7fa;">
    <td><b>Build N°</b></td><td>${env.BUILD_NUMBER}</td>
  </tr>
  <tr>
    <td><b>Statut</b></td>
    <td style="color:${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'};">
      <b>${currentBuild.currentResult}</b>
    </td>
  </tr>
  <tr style="background:#f5f7fa;">
    <td><b>Durée</b></td><td>${currentBuild.durationString}</td>
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

            echo "=== Rapports envoyés à ${RECIPIENT_MAIL} ==="
        }

        success {
            echo "Pipeline DevSecOps terminé avec succès !"
        }

        failure {
            echo "Pipeline en échec — consulter les rapports ci-dessus"
        }
    }
}
