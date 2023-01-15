def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {
    agent any
    tools {
	    maven "maven3.8.7"
	    jdk "myjdk"
	}
    environment {
        registry = "techhouse/docker-jenkins"
        registryCredential = 'dockercred'
    }
    stages{
        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
        stage('Test'){
            steps {
                sh 'mvn test'
            }

        }

        stage('Checkstyle Analysis'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool 'sonar4'
            }
            steps {
               withSonarQubeEnv('sonar') {
                   sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
              }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("UploadArtifact") {
            steps{
                nexusArtifactUploader(
                  nexusVersion: 'nexus3',
                  protocol: 'http',
                  nexusUrl: '172.31.3.203:8081',
                  groupId: 'QA',
                  version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                  repository: 'boljibeatz',
                  credentialsId: 'nexuslogin',
                  artifacts: [
                    [artifactId: 'vproapp',
                     classifier: '',
                     file: 'target/vprofile-v2.war',
                     type: 'war']
                 ]
               )
            }
        }
        stage('Build App Image') {
          steps {
            script {
              dockerImage = docker.build registry + ":V$BUILD_NUMBER"
            }
          }
        }
        stage('Upload Image'){
          steps{
            script {
              docker.withRegistry('', registryCredential) {
                dockerImage.push("V$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }
        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:V$BUILD_NUMBER"
          }
        }

        stage('Kubernetes Deploy') {
          agent {label 'KOPS'}
            steps {
              sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }

    }
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#techhouse',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }

}