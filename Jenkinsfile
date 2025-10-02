pipeline {
  agent any
  tools {
    maven 'maven3'
    jdk 'jdk17'
  }

  stages {

    stage ('git checkout') {
        steps {
            git branch: 'main', url: 'https://github.com/devopsgcp/bankingapp.git'
        }
    }

    stage ('git compile') {
        steps {
            sh 'mvn compile'
        }
    }

    stage ('git packkage') {
        steps {
            sh 'mvn package'
        }
    }

    stage ('trivy fs scan') {
        steps {
            sh 'mvn compile'
        }
    }








  }



}