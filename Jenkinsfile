pipeline {
  agent any

  tools {
    jdk 'jdk17'
    maven 'maven3'
  }

  environment {
    // Ensure 'mysonar' is defined in Jenkins Global Tool Configuration (SonarScanner)
    SCANNER_HOME = tool 'mysonar'
  }

  stages {
    stage('Git Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/RaviVarma06/Boardgame.git'
      }
    }

    stage('Compile') {
      steps {
        sh 'mvn -B -U compile'
      }
    }

    stage('Test') {
      steps {
        sh 'mvn -B test'
      }
    }

    stage('File System Scan') {
      steps {
        sh 'trivy fs --format table -o trivy-fs-report.html .'
      }
    }

    stage('SonarQube Analysis') {
      steps {
        // Use your configured SonarQube server name in Jenkins > Manage Jenkins > System > SonarQube Servers
        withSonarQubeEnv('sonar-server') {
          sh """
            \${SCANNER_HOME}/bin/sonar-scanner \
              -Dsonar.projectName=BoardGame \
              -Dsonar.projectKey=BoardGame \
              -Dsonar.java.binaries=.
          """
        }
      }
    }

    stage('Quality Gates') {
      steps {
        // Uses token stored in Jenkins Credentials with ID 'sonar-token'
        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B -DskipTests=false package'
      }
    }

    stage('Publish To Nexus') {
      steps {
        // Ensure 'global-settings' contains your Nexus <server> with credentials binding
        withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
          sh 'mvn -B deploy'
        }
      }
    }

    stage('Docker Image Scan') {
      steps {
        // Ensure the image exists locally or in the registry before scanning
        sh 'trivy image --format table -o trivy-image-report.html adijaiswal/boardshack:latest'
      }
    }

    stage('Push Docker Image') {
      steps {
        script {
          withDockerRegistry(credentialsId: 'dockerhub-cred', toolName: 'docker') {
            sh 'docker push ravi/boardshack:latest'
          }
        }
      }
    }

    stage('Deploy To Kubernetes') {
      steps {
        withKubeConfig(
          caCertificate: '',
          clusterName: 'kubernetes',
          contextName: '',
          credentialsId: 'k8-cred',
          namespace: 'webapps',
          restrictKubeConfigAccess: false,
          serverUrl: 'https://52.23.186.133:6443'
        ) {
          sh 'kubectl apply -f deployment-service.yaml'
        }
      }
    }

    stage('Verify the Deployment') {
      steps {
        withKubeConfig(
          caCertificate: '',
          clusterName: 'kubernetes',
          contextName: '',
          credentialsId: 'k8-cred',
          namespace: 'webapps',
          restrictKubeConfigAccess: false,
          serverUrl: 'https://52.23.186.133:6443'
        ) {
          sh 'kubectl get pods -n webapps'
          sh 'kubectl get svc -n webapps'
        }
      }
    }
  }

  post {
    always {
      script {
        def jobName       = env.JOB_NAME
        def buildNumber   = env.BUILD_NUMBER
        def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
        def bannerColor   = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

        def body = """
          <html>
            <body>
              <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                  <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
              </div>
            </body>
          </html>
        """

        emailext(
          subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
          body: body,
          to: 'ravivarma31m@gmail.com',
          from: 'jenkins@example.com',
          replyTo: 'jenkins@example.com',
          mimeType: 'text/html',
          attachmentsPattern: 'trivy-image-report.html'
        )
      }
    }
  }
}
``
