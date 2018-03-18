pipeline {
  agent any
  def server = Artifactory.server "SERVER_ID"
  def rtMaven = Artifactory.newMavenBuild()
  def buildInfo
  stages {
    stage('Build') {
      agent {
        docker {
          image 'maven:3-alpine'
          args '-v /home/bitwiseman/docker/.m2:/root/.m2'
        }
      }

      steps {
        sh 'mvn -B -DskipTests clean package'
        stash name: 'war', includes: 'target/**'
      }
    }
    stage('Backend') {
      agent {
        docker {
          image 'maven:3-alpine'
          args '-v /home/bitwiseman/docker/.m2:/root/.m2'
        }
      }

      steps {
        parallel(
         'Unit' : {
           unstash 'war'
           sh 'mvn -B -DtestFailureIgnore test || exit 0'
           junit '**/surefire-reports/**/*.xml'
          },
          'Performance' : {
            unstash 'war'
            sh '# ./mvn -B gatling:execute'
            sh 'mvn -'
         })
       }
    }
    stage('Frontend') {
      agent {
        docker 'node:alpine'
      }
      steps {
        sh 'node --version'
        sh '# yarn install'
        sh '# yarn global add gulp-cli'
        sh '# gulp test'
      }
    }
    stage('Static Analysis') {
      steps {
        sh 'echo Static'
      }
    }

    stage('Artifactory configuration') {
        // Tool name from Jenkins configuration
        rtMaven.tool = "Maven-3.5.0"
        // Set Artifactory repositories for dependencies resolution and artifacts deployment.
        rtMaven.deployer releaseRepo:'libs-release-local', snapshotRepo:'libs-snapshot-local', server: server
        rtMaven.resolver releaseRepo:'libs-release', snapshotRepo:'libs-snapshot', server: server
    }

    stage('Maven build') {
        buildInfo = rtMaven.run pom: 'jhipster-sample-application/pom.xml', goals: 'clean install'
    }

    stage('Publish build info') {
        server.publishBuildInfo buildInfo
    }

    stage('Deploy to Staging') {
      when {
        branch 'master'
      }
      steps {
        sh './deploy.sh staging'
        sh 'echo Notifying the Team!'
      }
    }

    stage('Deploy to Production') {
      when {
        branch 'master'
      }
      steps {
        input message: 'Deploy to production?',
                   ok: 'Fire away!'
        sh './deploy.sh production'
        sh 'echo Notifying the Team!'
      }
    }
  }
}
