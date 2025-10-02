pipeline {
  agent any
  tools {
    maven 'maven3'
    jdk 'jdk17'
  }
  environment {
    SCANNER_HOME = tool 'sonar-scanner'
  }

  stages {

    stage ('git checkout') {
        steps {
            git branch: 'main', url: 'https://github.com/devopsgcp/bankingapp.git'
        }
    }

    stage ('trivy fs scan') {
        steps {
            sh 'trivy fs --severity HIGH,CRITICAL --format json -o trivy-scan-report.json .'
        }
    }
   /* 
    stage ('owasp dependencies check') {
        steps {
            dependencyCheck additionalArguments: '--scan .', odcInstallation: 'DC'
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        }
    }
   */
     stage ('Sonarqube-scanner') {
        steps {
           withSonarQubeEnv('sonar-server') {
            sh '''
                $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=bankapp \
                -Dsonar.projectName=bankapp -Dsonar.sources=. \
                -Dsonar.java.binaries=target
            '''
           }
        }
    }

    stage ('Quality gate check') {
	  steps {
	    timeout(time: 1, unit: 'HOURS') {
		  waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
   
       }
     }
    }
    stage ('git packkage') {
      steps {
        sh 'mvn package'
      }
    }

    stage ('Push_to_nexus') {
	   steps {
	     withMaven(globalMavenSettingsConfig: 'bankapp', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
            sh 'mvn deploy'
         }
   
      }
   }



  }



}