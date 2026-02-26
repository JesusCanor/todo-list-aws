pipeline {
    agent any

    post {
        always {
            deleteDir()
        }
    }

    stages {
        stage('CI') {
            when { expression { env.GIT_BRANCH == 'origin/develop' } }

            stages {

//                stage('Get Code') {
//                    steps {
//                        withCredentials([usernamePassword(credentialsId: 'CP14',
//                                  usernameVariable: 'GIT_USER',
//                                passwordVariable: 'GIT_TOKEN')]) {
//                          sh '''
//                              set -eux
//                              git config user.name "JesusCanor"
//                              git config user.email "jenkins-cd@local"////                              git remote set-url origin https://${GIT_TOKEN}@github.com/JesusCanor/todo-list-aws.git
//                              git fetch origin
//                              git checkout -B CD origin/develop
//                      '''
//                      }
//                    }
//                }

                stage('Static and Security Tests') {
                    parallel {
                        stage('Static') {
                            steps {
                                sh '''
                                    python3 -m flake8 ./src --format=pylint > flake8.out
                                '''

                                archiveArtifacts artifacts: 'flake8.out', fingerprint: true

                            }
                        } //Stage static

                        stage('Security') {
                            steps {
                                sh '''
                                    python3 -m bandit -r ./src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                                '''
                                archiveArtifacts artifacts: 'bandit.out', fingerprint: true
                            }
                        }
                    }//parallel
                }//stage static and security

                stage('Deploy') {
                    steps {
                        sh '''
                            export AWS_REGION=us-east-1
                            export AWS_DEFAULT_REGION=us-east-1

                            set -eux

                            sam build
                            sam validate

                            sam deploy --config-env staging --resolve-s3 --no-fail-on-empty-changeset
                        '''
                        script {
                            env.BASE_URL = sh(
                                script: '''
                                    aws cloudformation describe-stacks \
                                    --stack-name todo-list-aws-staging \
                                    --region us-east-1 \
                                    --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                                    --output text
                                ''',
                                returnStdout: true
                                ).trim();

                                echo "BASE_URL raw => '${env.BASE_URL}'"
                    
                            if (!env.BASE_URL) {
                                error("Base_URL vacio.")
                            }
                        }
                    }
                } //Stage Deploy

                stage('Rest Test') {
                    steps {
                        sh '''
                            pytest -s --junitxml=result-rest.xml test/integration/todoApiTest.py
                        '''

                        junit 'result-rest.xml'

                    }
                }

                stage('Promote') {
                    steps {

                        withCredentials([usernamePassword(credentialsId: 'CP14',
                                  usernameVariable: 'GIT_USER',
                                  passwordVariable: 'GIT_TOKEN')]) {
                            sh '''
                                set -eux
                                git config user.name "${GIT_USER}"
                                git config user.email "jenkins-ci@local"

                                git remote set-url origin https://${GIT_TOKEN}@github.com/JesusCanor/todo-list-aws.git

                                git fetch origin
                                
                                git checkout -B master origin/master
                                git merge --no-ff -X theirs origin/develop -m "Promote develop to Master"


                                git push origin master
                            '''
                        }
                    }
                }
            }//stages CI
        }//CI

        stage('CD') {
            when { expression { env.GIT_BRANCH == 'origin/master' } }

            stages {
//                stage('Get Code') {
//                    steps {
//                        withCredentials([usernamePassword(credentialsId: 'CP14',
//                                  usernameVariable: 'GIT_USER',
//                                  passwordVariable: 'GIT_TOKEN')]) {
//                            sh '''
//                                set -eux
//                                git config user.name "JesusCanor"
//                                git config user.email "jenkins-cd@local"
//
//                                git remote set-url origin https://${GIT_TOKEN}@github.com/JesusCanor/todo-list-aws.git
//                                git fetch origin
//                                git checkout -B CD origin/master
//                        '''
//                        }
//                    }
//                }

                stage('Deploy') {
                    steps {
                        sh '''
                            set -eux
                            export AWS_DEFAULT_REGION=us-east-1

                            sam build
                            sam validate

                            sam deploy --config-env production --resolve-s3 --no-fail-on-empty-changeset
                        '''
                    }
                } //Stage Deploy

                stage('Rest Test') {
                    steps {
                        script {
                            env.BASE_URL = sh(
                                script: '''
                                    aws cloudformation describe-stacks \
                                    --stack-name todo-list-aws-production \
                                    --region us-east-1 \
                                    --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                                    --output text
                                ''',
                                returnStdout: true
                            ).trim()

                            echo "BASE_URL raw => '${env.BASE_URL}'"

                            if (!env.BASE_URL || env.BASE_URL == 'None') {
                                error("BASE_URL vacio.")
                            }
                        }

                        sh '''
                            python3 -m pytest -m "readonly" test/integration/todoApiTest.py
                        '''
                    }
                }
            }//stages CD
        }//CD
    }//Stages
}//pipeline
