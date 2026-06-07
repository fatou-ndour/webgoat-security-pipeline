pipeline {

    agent any

    triggers {
        githubPush()
    }

    environment {
        REPORT_DIR  = "${WORKSPACE}\\reports"
        MAIL_TO     = "fatoundour@esp.sn"
        TARGET_URL  = "http://localhost:8080/WebGoat"
        DC_BIN      = "C:\\dependency-check\\bin\\dependency-check.bat"
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    stages {

        stage('Checkout WebGoat') {
            steps {
                echo '=== TACHE 1 - Clonage de WebGoat ==='
                git url: 'https://github.com/WebGoat/WebGoat.git',
                    branch: 'main'
                bat "if not exist \"%REPORT_DIR%\" mkdir \"%REPORT_DIR%\""
                echo 'WebGoat clone avec succes'
            }
        }

        stage('SAST - Bearer CLI') {
            steps {
                echo '=== TACHE 1 - SAST : Bearer CLI ==='
                bat """
                    docker run --rm ^
                      -v "%WORKSPACE%:/src" ^
                      bearer/bearer:latest scan /src ^
                      --format json ^
                      --output /src/reports/bearer-report.json ^
                      --exit-code 0 ^
                      --scanner sast,secrets
                """
                echo 'SAST Bearer termine - rapport : reports/bearer-report.json'
            }
        }

        stage('SCA - Dependency-Check') {
            steps {
                echo '=== TACHE 1 - SCA : OWASP Dependency-Check ==='
                bat """
                    "%DC_BIN%" ^
                      --project "WebGoat-SCA" ^
                      --scan "%WORKSPACE%" ^
                      --format HTML ^
                      --format JSON ^
                      --out "%REPORT_DIR%" ^
                      --failOnCVSS 7 ^
                      --enableExperimental
                """
                echo 'SCA termine - rapport : reports/dependency-check-report.html'
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
                echo '=== Deploiement WebGoat pour scan DAST ==='
                bat """
                    docker rm -f webgoat-test 2>nul || echo Pas de container precedent
                    docker run -d ^
                        --name webgoat-test ^
                        -p 8080:8080 ^
                        -e TZ=Europe/Paris ^
                        webgoat/webgoat:latest
                """
                echo 'Attente du demarrage de WebGoat 45 secondes...'
                bat "timeout /t 45 /nobreak"
                echo 'WebGoat pret sur http://localhost:8080/WebGoat'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                echo '=== TACHE 1 - DAST : OWASP ZAP ==='
                bat """
                    docker run --rm ^
                      --network host ^
                      -v "%REPORT_DIR%:/zap/wrk" ^
                      ghcr.io/zaproxy/zaproxy:stable ^
                      zap-baseline.py ^
                      -t "%TARGET_URL%" ^
                      -r zap-report.html ^
                      -J zap-report.json ^
                      -l WARN ^
                      --auto
                """
                echo 'DAST ZAP termine - rapport : reports/zap-report.html'
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing         : true,
                        alwaysLinkToLastBuild: true,
                        keepAll              : true,
                        reportDir            : 'reports',
                        reportFiles          : 'zap-report.html',
                        reportName           : 'DAST - OWASP ZAP'
                    ])
                    bat "docker rm -f webgoat-test 2>nul || echo done"
                }
            }
        }

    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**/*',
                             allowEmptyArchive: true

            echo '=== TACHE 3 - Envoi du rapport par email ==='

            emailext(
                subject: "[Jenkins] WebGoat Security Scan - Build #${BUILD_NUMBER} - ${currentBuild.result ?: 'RUNNING'}",
                to: "${MAIL_TO}",
                mimeType: 'text/html',
                attachmentsPattern: 'reports/bearer-report.json,reports/zap-report.json',

                body: """
<html>
<body style="margin:0;padding:0;background:#f0f2f5;font-family:Arial,sans-serif;">
<table width="100%" cellpadding="0" cellspacing="0">
<tr><td align="center" style="padding:30px 16px;">
<table width="620" cellpadding="0" cellspacing="0"
       style="background:#fff;border-radius:8px;
              box-shadow:0 2px 8px rgba(0,0,0,0.12);overflow:hidden;">

  <tr>
    <td style="background:#1e2a3a;padding:22px 28px;">
      <div style="color:#fff;font-size:20px;font-weight:bold;">
        Rapport de Securite Jenkins
      </div>
      <div style="color:#7a8fa6;font-size:13px;margin-top:4px;">
        Pipeline CI/CD - WebGoat Security Scan
      </div>
    </td>
  </tr>

  <tr>
    <td style="background:${currentBuild.result == 'SUCCESS' ? '#1a7f37' : '#842029'};
               padding:13px 28px;text-align:center;">
      <span style="color:#fff;font-weight:bold;font-size:15px;">
        ${currentBuild.result == 'SUCCESS' ? 'BUILD REUSSI' : 'BUILD ECHOUE - VULNERABILITES DETECTEES'}
      </span>
    </td>
  </tr>

  <tr><td style="padding:20px 28px 10px;">
    <table width="100%" style="font-size:13px;border-collapse:collapse;">
      <tr style="background:#f8f9fa;">
        <td style="padding:8px 12px;color:#666;">Job</td>
        <td style="padding:8px 12px;font-weight:bold;">${JOB_NAME}</td>
        <td style="padding:8px 12px;color:#666;">Build</td>
        <td style="padding:8px 12px;font-weight:bold;">#${BUILD_NUMBER}</td>
      </tr>
      <tr>
        <td style="padding:8px 12px;color:#666;">Projet cible</td>
        <td style="padding:8px 12px;" colspan="3">WebGoat/WebGoat (Java - OWASP)</td>
      </tr>
      <tr style="background:#f8f9fa;">
        <td style="padding:8px 12px;color:#666;">Commit</td>
        <td style="padding:8px 12px;font-family:monospace;" colspan="3">
          ${env.GIT_COMMIT?.take(8) ?: 'N/A'} sur ${env.GIT_BRANCH ?: 'main'}
        </td>
      </tr>
    </table>
  </td></tr>

  <tr><td style="padding:10px 28px 20px;">
    <div style="font-size:14px;font-weight:bold;color:#1e2a3a;
                border-bottom:2px solid #e9ecef;padding-bottom:8px;margin-bottom:12px;">
      Resultats des 3 scans de securite
    </div>

    <div style="background:#e8f4fd;border-left:4px solid #1a73e8;
                padding:12px 16px;border-radius:0 4px 4px 0;margin-bottom:10px;">
      <strong>SAST - Bearer CLI</strong><br>
      <span style="font-size:13px;color:#444;">
        Analyse statique du code Java - Secrets - OWASP Top 10<br>
        Fichier joint : bearer-report.json
      </span>
    </div>

    <div style="background:#fef9e7;border-left:4px solid #f39c12;
                padding:12px 16px;border-radius:0 4px 4px 0;margin-bottom:10px;">
      <strong>SCA - OWASP Dependency-Check</strong><br>
      <span style="font-size:13px;color:#444;">
        CVE dans les dependances Maven de WebGoat<br>
        Rapport HTML disponible dans Jenkins
      </span>
    </div>

    <div style="background:#f5eefb;border-left:4px solid #9b59b6;
                padding:12px 16px;border-radius:0 4px 4px 0;margin-bottom:10px;">
      <strong>DAST - OWASP ZAP</strong><br>
      <span style="font-size:13px;color:#444;">
        Scan dynamique sur WebGoat en execution<br>
        Fichier joint : zap-report.json
      </span>
    </div>
  </td></tr>

  <tr><td style="padding:0 28px 24px;text-align:center;">
    <a href="${BUILD_URL}"
       style="background:#1e2a3a;color:#fff;padding:11px 28px;
              text-decoration:none;border-radius:5px;
              font-size:14px;font-weight:bold;">
      Voir le build complet sur Jenkins
    </a>
  </td></tr>

  <tr><td style="background:#f8f9fa;padding:12px 28px;
                 border-top:1px solid #e9ecef;font-size:11px;color:#999;">
    Genere automatiquement par Jenkins apres chaque git push
  </td></tr>

</table>
</td></tr>
</table>
</body>
</html>
                """
            )
        }
        success { echo 'Tous les scans termines sans vulnerabilite critique' }
        failure { echo 'Vulnerabilites critiques detectees - voir les rapports' }
    }
}
