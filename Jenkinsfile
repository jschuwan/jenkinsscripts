pipeline {
  agent any
  stages {
    stage('Test') {
      when {
        branch 'feature/*'
      }
      steps {
        withMaven() {
          cleanWs()
          sh 'test'
        }

      }
    }

    stage('Build') {
      when {
        branch 'main'
      }
      steps {
        withMaven() {
          sh 'mvn -f api1Reimb/pom.xml clean install'
          sh 'mvn -f api1Reimb/pom.xml clean package -DskipTests'
        }

      }
    }

    stage('Docker Build') {
      when {
        branch 'main'
      }
      steps {
        dir(path: 'api1Reimb/') {
          script {
            echo "$registry:$currentBuild.number"
            dockerImage = docker.build "$registry"
          }

        }

      }
    }

    stage('Docker Deliver') {
      when {
        branch 'main'
      }
      steps {
        script {
          docker.withRegistry('', dockerHubCreds) {
            dockerImage.push("$currentBuild.number")
            dockerImage.push("latest")
          }
        }

      }
    }

    stage('Wait for approval') {
      when {
        branch 'main'
      }
      steps {
        script {
          try {
            timeout(time: 1, unit: 'MINUTES') {
              approved = input message: 'Deploy to production?', ok: 'Continue',
              parameters: [choice(name: 'approved', choices: 'Yes\nNo', description: 'Deploy build to production')]
              if(approved != 'Yes') {
                error('Build did not pass approval')
              }
            }
          } catch(error) {
            error('Build failed because timeout was exceeded')
          }
        }

      }
    }

    stage('Deploy to GKE') {
      when {
        branch 'main'
      }
      steps {
        sh 'sed -i "s/%TAG%/$BUILD_NUMBER/g" ./api1Reimb/k8s/deployment.yml'
        sh 'cat ./api1Reimb/k8s/deployment.yml'
        step([$class: 'KubernetesEngineBuilder',
                                projectId: 'spherical-gate-338602',
                                clusterName: 'spherical-gate-338602-gke',
                                zone: 'us-central1',
                                manifestPattern: 'api1Reimb/k8s/',
                                credentialsId: 'spherical-gate-338602',
                                verifyDeployments: true
                            ])
      }
    }

  }
  environment {
    registry = 'jschuwan/api1'
    dockerHubCreds = 'docker_hub'
    dockerImage = ''
    deploymentFile = 'api1Reimb/k8s/deployment.yml'
  }
}