pipeline {
    agent {
        docker {
            image 'maven:3.8.4-jdk-11'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    environment {
        CI_SERVER = 'Jenkins'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
                sh 'mvn -B checkstyle:checkstyle spotbugs:spotbugs pmd:pmd pmd:cpd'
                mineRepository()
            }
        }
        stage('Test') {
            steps {
                sh 'mvn -B -Dmaven.test.failure.ignore=true verify'
            }
        }
        stage('Deploy - Staging') {
            steps {
                milestone label: "Deployment Stages", ordinal: 1
                script {
                    shortCommit = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
                    build([
                        job: 'Tests/Deployment to Staging',
                        parameters: [string(name: 'VERSION', value: shortCommit)],
                        propagate: true,
                        wait: true
                    ])
                }
            }
        }
        stage('Sanity check') {
            steps {
                input "Does the staging environment look ok?"
                milestone label: "Sanity check", ordinal: 2
            }
        }
        stage('Deploy - Production') {
            when {
                branch 'release-*'
            }
            steps {
                script {
                    shortCommit = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
                    build([
                        job: 'Tests/Deployment to Production',
                        parameters: [string(name: 'VERSION', value: shortCommit)],
                        propagate: true,
                        wait: true
                    ])
                }
            }
        }
    }
    post {
        always {
            junit 'target/surefire-reports/*.xml'
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            recordIssues(tools: [checkStyle(), pmdParser(), cpd(), spotBugs(useRankAsPriority: true)])
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'target/site/jacoco', reportFiles: 'index.html', reportName: 'JaCoCo HTML Report', reportTitles: ''])
        }
        cleanup {
            deleteDir()
        }
    }
}
