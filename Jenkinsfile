pipeline {
    agent any
    environment {
        registryCredential = 'ecr:us-east-1:awsjenkinsiam'
        appRegistry = "179426988817.dkr.ecr.us-east-1.amazonaws.com/hridak"
        hridakRegistry = "https://179426988817.dkr.ecr.us-east-1.amazonaws.com"
        cluster = "Hridak"
        service = "hridaksvc"
        taskfamily = "Hridak-TD"
        desiredcount = "2"
    }

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "MAVEN"
    }

    stages {
        stage('Build') {
            steps {
                // Get some code from a GitHub repository
                git branch: 'docker', url: 'https://github.com/devopswala/AWSL-S.git'

                // Run Maven on a Unix agent.
                sh "mvn clean install"

                // To run Maven on a Windows agent, use
                // bat "mvn -Dmaven.test.failure.ignore=true clean package"
            }
        }
		stage('Code Analysis with CheckStyle') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
		    post {
                success {
                    archiveArtifacts 'target/*.war'
                }
            }
        }
        stage('Sonar Scan') {
            environment {
                scannerHome = tool 'Sonar'
            }
            steps {
                withSonarQubeEnv('SonarServer') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=visualpathit \
                    -Dsonar.projectName=visualpathit \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.checkstyle.reportsPath=target/checkstyle-result.xml'''
                }
                timeout( time: 10, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        // stage('Publish to Nexus Artifactory') {
        //     steps {
        //     //
        //     nexusArtifactUploader artifacts: [[artifactId: 'hridakapp', classifier: '', file: 'target/hridak-v2.war', type: 'war']], credentialsId: 'nexus', groupId: 'hridak-grp-repo', nexusUrl: '54.161.18.157:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'Hridak', version: '${BUILD_ID}'
        //         }
        // }
        stage ('Docker Build') {
            steps {
                script {
                dockerimage = docker.build( appRegistry + ":${BUILD_NUMBER}", "./Docker-files/app/multistage/")
                }
            }
        }
        stage ('Push DockerImage to ECR') {
            steps {
                script {
                docker.withRegistry( hridakRegistry, registryCredential ){
                    dockerimage.push()
                }
                }
            }
        }
        stage ('Deploy app to ECS') {
            steps {
                withAWS(credentials: 'awsjenkinsiam', region: 'us-east-1') {
                    sh 'aws ecs update-service --cluster ${cluster} --service ${service} --task-definition ${taskfamily}:1 --desired-count ${desiredcount}'
                }
            }
        }
        stage('Notify Slack') {
            steps {
                echo 'Notifying Slack'
                slackSend channel: '#jenkinscicd',
                    color: '#439FE0',
                    message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n For More information see: ${env.BUILD_URL}"

            }
        }
    }
}
