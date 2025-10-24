pipeline {
    agent any

    kubernetes {
        yaml '''
apiVersion: v1
kind: Pod
spec:
    containers:
        - name: gitleaks
          image: zricethezav/gitleaks:latest
          command:
        - sleep
        args:
        - 99d
'''
    }

    tools {
        maven 'maven3'
    }

    environment {
        APP_NAME = 'WebGoat'
        SONARQUBE_SERVER = 'sonarqube'
        BUILD_TIMESTAMP = sh(script: "date '+%Y-%m-%d_%H-%M-%S'",returnStdout:true).trim()
    }

    //STAGES
    stages {

        stage('Checkout') {
            steps {
                echo 'STAGE: Checkout Source Code'
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Build: ${env.BUILD_NUMBER}"

                sh '''
                                echo "Working on Directory: $(pwd)"
                                echo "Git Status"
                                git status
                                echo "Git Log (last commit)"
                                git log -1
                                '''
            }
        }

        stage('Secret Scanning') {
            steps {
                echo "Stage: Secret Scanning"
                container('gitleaks') {
                    script {
                        def exitCode = sh(
                            script: 'gitleaks detect --source=. --report-format=json --report-path=gitleaks-report.json --no-git',
                            returnStatus: true
                        )

                        if (exitCode == 0) {
                            echo "No secrets found"
                        } else if (exitCode == 1) {
                            error 'SECRET DETECTED! Check gitleaks-report.json'
                        } else {
                            error "Gitleaks failed with exit code: ${exitCode}"
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Build') {
            steps {
                echo 'Stage: Building Application'
                sh '''
                                mvn clean package -DskipTests
                                '''
                echo 'Build completed'
            }
        }

        stage('Unit Test') {
            steps {
                echo 'Stage: Running Unit Test'
                sh '''
                            mvn test
                            '''
                echo 'Unit test passed'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }

        }
        stage('Parallel Scans') {

            parallel {
                stage('SCA - Dependency Check') {
                    steps {
                        echo "Parallel Stage: SCA"

                        dependencyCheck additionalArguments: '--scan ./ --format HTML --format XML', odcInstallation: 'OWASP SCA'
                        dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    }
                }
                stage('SAST - SonarQube') {
                    steps {
                        echo "Parallel Stage: SAST"

                        script {
                            withSonarQubeEnv("${SONARQUBE_SERVER}") {
                                sh '''
                                                        mvn sonar:sonar \
                                                        -Dsonar.projectKey=${APP_NAME} \
                                                        -Dsonar.projectName=${APP_NAME} \
                                                        -Dsonar.javabinaries=target/classes
                                                        '''
                            }

                            echo "SAST scan completed"
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo "Stage: Waiting for SonarQube Quality Gate"

                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Quality Gate failed: ${qg.status}"
                        }
                    }
                }
                echo 'Quality Gate passed'
            }
        }

        stage('Package') {
                steps {
                        echo "Stage: Package Application"
                        sh '''
                        echo "Packagin WebGoat Jar...."
                        ls -lh target/*.jar
                        '''
                        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                        echo "Package completed"
                }
        }

        stage('Deploy to Dev') {
                steps {
                        echo "Stage: Deploy to Development Enviorment"
                        //Not doing anything just wanted to have this one to look good in the Jenkins job.script

                }
        }

        post {
                always {
                        echo "pipeline completed"
                        cleanWs()
                }
                success {
                        echo "Pipeline completed succesfully."

                }
                failure {
                        echo "Pipeline Failed"
                }
        }
    }
}
