pipeline {
    agent any

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/bvbvcvrg/juice-shop-exam.git',
                    credentialsId: 'identifiants-github'
            }
        }

        stage('Build / Preparation') {
            steps {
                sh '''
                    node --version
                    npm --version
                '''
            }
        }

        stage('Security Analysis') {
            steps {
                sh '''
                    bearer scan . --scanner=sast,secrets --format html --output bearer-report.html --exit-code 0
                    bearer scan . --scanner=sast,secrets --format json --output bearer-report.json --exit-code 0
                '''
            }
        }

        stage('Additional Security Check') {
            steps {
                sh '''
                    npm audit --json > npm-audit-report.json || true
                    npm audit > npm-audit-report.txt || true
                '''
            }
        }

        stage('Report Generation') {
            steps {
                archiveArtifacts artifacts: 'bearer-report.html, bearer-report.json, npm-audit-report.txt, npm-audit-report.json', fingerprint: true
            }
        }
    }

    post {
        always {
            echo "Pipeline termine avec le statut : ${currentBuild.currentResult}"
        }
    }
}
