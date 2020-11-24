pipeline {
  agent any
  stages {
    stage('SCM') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      parallel {
        stage('Compile') {
          agent {
            docker {
              image 'maven:3.6.0-jdk-8-alpine'
              args '-v /root/.m2/repository:/root/.m2/repository'
              reuseNode true
            }

          }
          steps {
            sh ' mvn clean compile'
          }
        }

        stage('CheckStyle') {
          agent {
            docker {
              image 'maven:3.6.0-jdk-8-alpine'
              args '-v /root/.m2/repository:/root/.m2/repository'
              reuseNode true
            }

          }
          steps {
            sh ' mvn checkstyle:checkstyle'
            recordIssues(enabledForFailure: true, tool: checkStyle(pattern: '**/target/checkstyle-result.xml'))
          }
        }

      }
    }

    stage('Unit Tests') {
      agent {
        docker {
          image 'maven:3.6.0-jdk-8-alpine'
          args '-v /root/.m2/repository:/root/.m2/repository'
          reuseNode true
        }

      }
      when {
        anyOf {
          branch 'master'
          branch 'develop'
        }

      }
      post {
        always {
          junit 'target/surefire-reports/**/*.xml'
        }

      }
      steps {
        sh 'mvn test'
      }
    }

    stage('Integration Tests') {
      agent {
        docker {
          image 'maven:3.6.0-jdk-8-alpine'
          args '-v /root/.m2/repository:/root/.m2/repository'
          reuseNode true
        }

      }
      when {
        anyOf {
          branch 'master'
          branch 'develop'
        }

      }
      post {
        always {
          junit 'target/failsafe-reports/**/*.xml'
        }

        success {
          stash(name: 'artifact', includes: 'target/*.war')
          stash(name: 'pom', includes: 'pom.xml')
          archiveArtifacts 'target/*.war'
        }

      }
      steps {
        sh 'mvn verify -Dsurefire.skip=true'
      }
    }

    stage('Code Quality Analysis') {
      post {
        always {
          recordIssues(aggregatingResults: true, tools: [mavenConsole(), java(), javaDoc(), checkStyle(pattern: '**/target/checkstyle-result.xml'), findBugs(pattern: '**/target/findbugsXml.xml', useRankAsPriority: true), pmdParser(pattern: '**/target/pmd.xml')])
        }

      }
      parallel {
        stage('PMD') {
          agent {
            docker {
              image 'maven:3.6.0-jdk-8-alpine'
              args '-v /root/.m2/repository:/root/.m2/repository'
              reuseNode true
            }

          }
          steps {
            sh ' mvn pmd:pmd'
          }
        }

        stage('Findbugs') {
          agent {
            docker {
              image 'maven:3.6.0-jdk-8-alpine'
              args '-v /root/.m2/repository:/root/.m2/repository'
              reuseNode true
            }

          }
          steps {
            sh ' mvn findbugs:findbugs'
          }
        }

        stage('JavaDoc') {
          agent {
            docker {
              image 'maven:3.6.0-jdk-8-alpine'
              args '-v /root/.m2/repository:/root/.m2/repository'
              reuseNode true
            }

          }
          steps {
            sh ' mvn javadoc:javadoc'
          }
        }

        stage('SonarQube') {
          agent {
            docker {
              image 'maven:3.6.0-jdk-8-alpine'
              args '-v /root/.m2/repository:/root/.m2/repository'
              reuseNode true
            }

          }
          steps {
            sh " mvn sonar:sonar -Dsonar.host.url=$SONARQUBE_URL:$SONARQUBE_PORT -Dsonar.login=$SONARQUBE_TOKEN"
          }
        }

      }
    }

    stage('Deploy Artifact To Nexus') {
      when {
        anyOf {
          branch 'master'
          branch 'develop'
        }

      }
      steps {
        script {
          unstash 'pom'
          unstash 'artifact'
          // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
          pom = readMavenPom file: "pom.xml";
          // Find built artifact under target folder
          filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
          // Print some info from the artifact found
          echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
          // Extract the path from the File found
          artifactPath = filesByGlob[0].path;
          // Assign to a boolean response verifying If the artifact name exists
          artifactExists = fileExists artifactPath;
          if (artifactExists) {
            nexusArtifactUploader(
              nexusVersion: NEXUS_VERSION,
              protocol: NEXUS_PROTOCOL,
              nexusUrl: NEXUS_URL,
              groupId: pom.groupId,
              version: pom.version,
              repository: NEXUS_REPOSITORY,
              credentialsId: NEXUS_CREDENTIAL_ID,
              artifacts: [
                // Artifact generated such as .jar, .ear and .war files.
                [artifactId: pom.artifactId,
                classifier: '',
                file: artifactPath,
                type: pom.packaging
              ],
              // Lets upload the pom.xml file for additional information for Transitive dependencies
              [artifactId: pom.artifactId,
              classifier: '',
              file: "pom.xml",
              type: "pom"
            ]
          ]
        )
      } else {
        error "*** File: ${artifactPath}, could not be found";
      }
    }

  }
}

stage('Deploy to Staging Servers') {
  agent {
    docker {
      image 'ahmed24khaled/ansible-management'
      reuseNode true
    }

  }
  when {
    anyOf {
      branch 'master'
      branch 'develop'
    }

  }
  steps {
    script {
      pom = readMavenPom file: "pom.xml"
      repoPath = "${pom.groupId}".replace(".", "/") + "/${pom.artifactId}"
      version = pom.version
      artifactId = pom.artifactId
      withEnv(["ANSIBLE_HOST_KEY_CHECKING=False", "APP_NAME=${artifactId}", "repoPath=${repoPath}", "version=${version}"]) {
        sh '''

curl --silent "http://$NEXUS_URL/repository/maven-snapshots/${repoPath}/${version}/maven-metadata.xml" > tmp &&
egrep '<value>+([0-9\\-\\.]*)' tmp > tmp2 &&
tail -n 1 tmp2 > tmp3 &&
tr -d "</value>[:space:]" < tmp3 > tmp4 &&
REPO_VERSION=$(cat tmp4) &&

export APP_SRC_URL="http://${NEXUS_URL}/repository/maven-snapshots/${repoPath}/${version}/${APP_NAME}-${REPO_VERSION}.war" &&
ansible-playbook -v -i ./ansible_provisioning/hosts --extra-vars "host=staging" ./ansible_provisioning/playbook.yml

'''
      }
    }

  }
}

stage('Deploy to Production Servers') {
  agent {
    docker {
      image 'ahmed24khaled/ansible-management'
      reuseNode true
    }

  }
  when {
    branch 'master'
  }
  steps {
    script {
      pom = readMavenPom file: "pom.xml"
      repoPath = "${pom.groupId}".replace(".", "/") + "/${pom.artifactId}"
      version = pom.version
      artifactId = pom.artifactId
      withEnv(["ANSIBLE_HOST_KEY_CHECKING=False", "APP_NAME=${artifactId}", "repoPath=${repoPath}", "version=${version}"]) {
        sh '''

curl --silent "$NEXUS_URL/repository/maven-snapshots/${repoPath}/${version}/maven-metadata.xml" > tmp &&
egrep '<value>+([0-9\\-\\.]*)' tmp > tmp2 &&
tail -n 1 tmp2 > tmp3 &&
tr -d "</value>[:space:]" < tmp3 > tmp4 &&
REPO_VERSION=$(cat tmp4) &&

export APP_SRC_URL="http://${NEXUS_URL}/repository/maven-snapshots/${repoPath}/${version}/${APP_NAME}-${REPO_VERSION}.war" &&
ansible-playbook -v -i ./ansible_provisioning/hosts --extra-vars "host=production" ./ansible_provisioning/playbook.yml

'''
      }
    }

  }
}

}
environment {
NEXUS_VERSION = 'nexus2'
NEXUS_PROTOCOL = 'http'
NEXUS_URL = 'nexus:8081'
NEXUS_REPOSITORY = 'maven-snapshots'
NEXUS_CREDENTIAL_ID = 'nexus-credentials'
SONARQUBE_URL = 'http://192.168.99.100'
SONARQUBE_PORT = '9000'
SONARQUBE_TOKEN = '76e028134d617de20fb2a66664a3f995907984b9'
}
options {
skipDefaultCheckout()
}
}