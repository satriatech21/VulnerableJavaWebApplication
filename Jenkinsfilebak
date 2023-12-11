pipeline {
    agent none
    stages {
        stage('Maven Compile and SAST Spotbugs') {
            agent {
                label 'maven'
            }
            steps {
                sh 'mvn compile spotbugs:spotbugs'
                sh 'cp ./target/spotbugs.html ./spotbugs.html'
                sh 'cp ./target/spotbugsXml.xml ./spotbugsXml.xml'
                sh 'curl -X POST https://demo.defectdojo.org/api/v2/import-scan/ -H "Authorization: Token 548afd6fab3bea9794a41b31da0e9404f733e222" -F "scan_type=SpotBugs Scan" -F "file=@./spotbugsXml.xml;type=text/xml" -F "engagement=1"'
                archiveArtifacts artifacts: './spotbugs.html'
                archiveArtifacts artifacts: './spotbugsXml.xml'
            }
        }
        stage('Secret Scanning') {
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint='
                }
            }
            steps {
                sh 'trufflehog --no-update filesystem . --json > trufflehogscan.json'
                sh 'cat trufflehogscan.json'
                archiveArtifacts artifacts: 'trufflehogscan.json'
            }
        }
        stage('SCA') {
            agent {
                docker {
                    image 'owasp/dependency-check:latest'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint='
                }
            }
            steps {
                sh '/usr/share/dependency-check/bin/dependency-check.sh --scan . --project "VulnerableJavaWebApplication" --format ALL'
                archiveArtifacts artifacts: 'dependency-check-report.html'
                archiveArtifacts artifacts: 'dependency-check-report.json'
                archiveArtifacts artifacts: 'dependency-check-report.xml'
            }
        }
        stage('Build Docker Image') {
            agent {
                docker {
                    image 'docker:dind'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'docker build -t vulnerable-java-application:0.1 .'
            }
        }
        stage('Run Docker Image') {
            agent {
                label 'built-in'
            }
            steps {
                sh 'docker rm --force vulnerable-java-application'
                sh 'docker run --name vulnerable-java-application -p 9000:9000 -d vulnerable-java-application:0.1'
            }
        }
        stage('DAST') {
            agent {
                docker {
                    image 'owasp/zap2docker-stable:latest'
                    args '-u root -v /var/run/docker.sock:/var/run/docker.sock --entrypoint= -v .:/zap/wrk/:rw'
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'zap-full-scan.py -t https://172.18.0.3:9000 -r zapfull.html -x zapfull.xml'
                }
                sh 'cp /zap/wrk/zapfull.html ./zapfull.html'
                sh 'cp /zap/wrk/zapfull.xml ./zapfull.xml'
                archiveArtifacts artifacts: 'zapfull.html'
                archiveArtifacts artifacts: 'zapfull.xml'
            }
        }
    }
    post {
        always {
            node('built-in') {
                sh 'curl -X POST https://demo.defectdojo.org/api/v2/import-scan/ -H "Authorization: Token 548afd6fab3bea9794a41b31da0e9404f733e222" -F "scan_type=Trufflehog Scan" -F "file=@./trufflehogscan.json;type=application/json" -F "engagement=1"'
                sh 'curl -X POST https://demo.defectdojo.org/api/v2/import-scan/ -H "Authorization: Token 548afd6fab3bea9794a41b31da0e9404f733e222" -F "scan_type=Dependency Check Scan" -F "file=@./dependency-check-report.xml;type=text/xml" -F "engagement=1"'
                sh 'curl -X POST https://demo.defectdojo.org/api/v2/import-scan/ -H "Authorization: Token 548afd6fab3bea9794a41b31da0e9404f733e222" -F "scan_type=ZAP Scan" -F "file=@./zapfull.xml;type=text/xml" -F "engagement=1"'
            }
        }
    }
}
