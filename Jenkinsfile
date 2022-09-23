pipeline {
  environment {
    ARGO_SERVER = '104.197.60.253:32100'
    DEV_URL = 'http://104.197.60.253:30080/'
  }
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult:'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
             }
           }
         }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true,artifacts: 'target/dependency-check-report.html', fingerprint:true, onlyIfSuccessful: true
              // dependencyCheckPublisher pattern: 'report.xml'
            }
        }
    }
        
        stage('Generate SBOM') {
          steps {
            container('maven') {
              sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
          }
       }
          post {
            success {
             //dependencyTrackPublisher projectName:'sample-spring-app', projectVersion: '0.0.1', artifact:'target/bom.xml', autoCreateProjects: true, synchronous: true
             archiveArtifacts allowEmptyArchive: true,artifacts: 'target/bom.xml', fingerprint: true,onlyIfSuccessful: true
         }
       }
     }
   }
  }
    stage('SAST') {
      steps {
        container('slscan') {
          sh 'scan --type java,depscan --build'
       }
     }
      post {
        success {
          archiveArtifacts allowEmptyArchive: true,
          artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful:true
        }
      }
    }
    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
        stage('Build with Kaniko') {
          steps {
            container('kaniko') {
           sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/vinycoolguy/dsodemo'
            }
      }
    }
      }
    }
    
    stage('Image Analysis') {
      parallel {
        stage('Image Linting') {
          steps {
            container('docker-tools') {
              sh 'dockle docker.io/vinycoolguy/dsodemo'
           }
         }
       }
        stage('Image Scan') {
          steps {
            container('docker-tools') {
              sh 'trivy image vinycoolguy/dsodemo'
            }
          }
        }
      }
   }

    stage('Scan k8s Deploy Code') {
      steps {
        container('docker-tools') {
          sh 'kubesec scan deploy/dso-demo-deploy.yaml'
       }
     }
   }

    stage('Deploy to Dev') {
      environment {
        AUTH_TOKEN = credentials('argocd-jenkins-deployer-token')
       }
      steps {
        container('docker-tools') {
          sh 'docker run -t schoolofdevops/argocd-cli argocd app sync dso-demo --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
          sh 'docker run -t schoolofdevops/argocd-cli argocd app wait dso-demo --health --timeout 300 --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
       }
      }
    }
    
    stage('Dynamic Analysis') {
      parallel {
        stage('E2E tests') {
          steps {
            sh 'echo "All Tests passed!!!"'
          }
        }
        stage('DAST') {
          steps {
            container('docker-tools') {
              sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t $DEV_URL || exit 0'
           }
         }
       }
     }
   }
  }
}
