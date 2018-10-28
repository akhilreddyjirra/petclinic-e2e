#!groovy

pipeline {
    agent { label 'sensen-build-slave-01' } 
//	agent {
//        docker {
//            image 'maven:3.5.4-jdk-8'
//            args '--network ci --mount type=volume,source=ci-maven-home,target=/root/.m2'
//        }
//    }

    environment {
        
        ORG_NAME = 'deors'
        APP_NAME = 'deors-demos-petclinic'
        APP_CONTEXT_ROOT = 'petclinic'
      //  TEST_CONTAINER_NAME = 'ci-${APP_NAME}-${BUILD_NUMBER}'
        TEST_CONTAINER_NAME = 'ci-${APP_NAME}'
      //	DOCKER_HOST = 'tcp://178.128.103.136:4243'
    }

    stages {
        stage('Compile') {
            steps {
                script {
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- compiling project -=-"
                sh "'${mvnHome}/bin/mvn' clean compile"
                }
           }     
        }

        stage('Unit tests') {
            steps {
                script {
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- execute unit tests -=-"
                sh "'${mvnHome}/bin/mvn' test"
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }

        stage('Mutation tests') {
            steps {
                script{
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- execute mutation tests -=-"
                sh "'${mvnHome}/bin/mvn' org.pitest:pitest-maven:mutationCoverage"
                }
            }
        }

        stage('Package') {
            steps {
                script{
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- packaging project -=-"
                sh "'${mvnHome}/bin/mvn' package -DskipTests"
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
            }
        }

        stage('Build Docker image') {
            steps {
                script {
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- build Docker image -=-"
                sh "'${mvnHome}/bin/mvn' docker:build"
                }
            }
        }

        stage('Run Docker image') {
            steps {
                script {
                echo "-=- run Docker image -=-"
                sh "docker run --name ${TEST_CONTAINER_NAME} --detach --rm --network ci -p 8080:8080 --expose 6300 --env JAVA_OPTS='-javaagent:/usr/local/tomcat/jacocoagent.jar=output=tcpserver,address=*,port=6300' ${ORG_NAME}/${APP_NAME}:latest"
                }     
            }
        }

//        stage('Integration tests') {
//            steps {
//                script {
//                def mvnHome = tool 'Maven 3.5.4'
//                echo "-=- execute integration tests -=-"
//                sh "'${mvnHome}/bin/mvn' failsafe:integration-test failsafe:verify -DargLine=\"-Dtest.selenium.hub.url=http://selenium-hub:4444/wd/hub -Dtest.target.server.url=http://${TEST_CONTAINER_NAME}:8080/${APP_CONTEXT_ROOT}\""
//                sh "java -jar target/dependency/jacococli.jar dump --address ${TEST_CONTAINER_NAME} --port 6300 --destfile target/jacoco-it.exec"
//                junit 'target/failsafe-reports/*.xml'
//                jacoco execPattern: 'target/jacoco-it.exec'
//                }
//            }
//        }

        stage('Performance tests') {
            steps {
                script {
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- execute performance tests -=-"
                sh " curl -v http://localhost:8080/petclinic"
                sh "'${mvnHome}/bin/mvn' jmeter:jmeter jmeter:results -Djmeter.target.host=${TEST_CONTAINER_NAME} -Djmeter.target.port=8080 -Djmeter.target.root=${APP_CONTEXT_ROOT}"
                perfReport sourceDataFiles: 'target/jmeter/results/*.csv', errorUnstableThreshold: 0, errorFailedThreshold: 5, errorUnstableResponseTimeThreshold: 'petclinic.jtl:100'
                }
            }
        }

        stage('Dependency vulnerability tests') {
            steps {
                script {
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- run dependency vulnerability tests -=-"
                sh "'${mvnHome}/bin/mvn' dependency-check:check"
                dependencyCheckPublisher failedTotalHigh: '30', unstableTotalHigh: '25', failedTotalNormal: '110', unstableTotalNormal: '100'
                }
            }
        }

        stage('Code inspection & quality gate') {
            steps {
                script {
                echo "-=- run code inspection & check quality gate -=-"
                withSonarQubeEnv('ci-sonarqube') {
                    sh "'${mvnHome}/bin/mvn' sonar:sonar"
                }
                timeout(time: 10, unit: 'MINUTES') {
                    //waitForQualityGate abortPipeline: true
                    script  {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK' && qg.status != 'WARN') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                           }
                        }
                    }
                }
            }
        }

        stage('Push Docker image') {
            steps {
                script {
                def mvnHome = tool 'Maven 3.5.4'
                echo "-=- push Docker image -=-"
                sh "'${mvnHome}/bin/mvn' docker:push"
                }
            }
        }
    }

    post {
        always {
            script {
            echo "-=- remove deployment -=-"
            sh "docker stop ${TEST_CONTAINER_NAME}"
            }
        }
    }
}
