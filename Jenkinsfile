pipeline {
 agent any
 environment {	   
  // This can be nexus3 or nexus2
  NEXUS_VERSION = "nexus3"
  // This can be http or https
  NEXUS_PROTOCOL = "http"

  // Where your Nexus is running. In my case:
  // NEXUS_URL = "ec2-52-212-29-159.eu-west-1.compute.amazonaws.com:8081"
	 // windows iponfig e tirar o ip da nossa maquina
   NEXUS_URL = "192.168.203.3:8081"
  // Repository where we will upload the artifact
  NEXUS_REPOSITORY = "maven-nexus-repo"
  // Jenkins credential id to authenticate to Nexus OSS
  NEXUS_CREDENTIAL_ID = "nexus_user"
  /* 
 Notice : I put as url nexus:8081 
 but I could also put 192.168.99.100 (localhost for Linux) 
 but since in our case Jenkins and Nexus are in the same network named “devops”, 
 so the resolution of DNS between the containers will be made automatically by Docker.
  i.e we can use either @ip, hostname (default service name) or the id of the container
  to ping/connect to the containers.
  */
  SONARQUBE_URL = "http://192.168.203.3" 
  SONARQUBE_PORT = "9000"
  SONARQUBE_TOKEN = "fd8bfb60348c7b145dd631dc4ff1d88990ddebbe"
 }
 options {
  skipDefaultCheckout()
 }
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
       // to use the same node and workdir defined on top-level pipeline for all docker agents
       reuseNode true
      }
     }
     steps {
        sh 'mvn clean compile'      
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
      recordIssues enabledForFailure: true, tool: checkStyle(pattern: '**/target/checkstyle-result.xml') 	  
	  
	  // no encontrou este plugin no jenkins, eu esquisei peo plugin e nao encontro pagina error 404
	  // https://github.com/jenkinsci/checkstyle-plugin This plugin reached end-of-life.
	  /* step([$class: 'CheckStylePublisher',
       //canRunOnFailed: true,
       defaultEncoding: '',
       healthy: '100',
       pattern: ' * * /target/checkstyle-result.xml',
       unHealthy: '90',
       //useStableBuildAsReference: true
      ]) */
     }
    }
   }
  }
  stage('Unit Tests') {
   when {
    anyOf { branch 'master'; branch 'develop' ; branch 'main'}
   }
   agent {
    docker {
     image 'maven:3.6.0-jdk-8-alpine'
     args '-v /root/.m2/repository:/root/.m2/repository'
     reuseNode true
    }
   }
   steps {
    sh 'mvn test'
   }
   post {
    always {
     junit 'target/surefire-reports/**/*.xml'
    }
   }
  }
  stage('Integration Tests') {
   when {
    anyOf { branch 'master'; branch 'develop'  ; branch 'main'}
   }
   agent {
    docker {
     image 'maven:3.6.0-jdk-8-alpine'
     args '-v /root/.m2/repository:/root/.m2/repository'
     reuseNode true
    }
   }
   steps {
    sh 'mvn verify -Dsurefire.skip=true'
   }
   post {
    always {
     junit 'target/failsafe-reports/**/*.xml'
    }
    success {
     stash(name: 'artifact', includes: 'target/*.war')
     stash(name: 'pom', includes: 'pom.xml')
     // to add artifacts in jenkins pipeline tab (UI)
     archiveArtifacts 'target/*.war'
    }
   }
  }
  stage('Code Quality Analysis') {
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
      // using pmd plugin
	  // recordIssues enabledForFailure: true, tool: pmdParser(pattern: '**/target/pmd.xml')
    
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
	  // recordIssues enabledForFailure: true, tool: findBugs(pattern: '**/target/findbugsXml.xml')
	   

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
	 // recordIssues enabledForFailure: true, tools: [mavenConsole(), java(), javaDoc()]
      
     }
    }
    stage('SonarQube') {
     agent {
      docker {
       image 'maven:3.6.0-jdk-8-alpine'
       args "-v /root/.m2/repository:/root/.m2/repository"
       reuseNode true
      }
     }
     steps {
      //  sh "mvn sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -Dsonar.branch=${env.BRANCH_NAME}"
      // sh "mvn sonar:sonar"

      // sh " mvn clean verify -P sonar  -Dsonar.login=$SONARQUBE_TOKEN"

	     sh " mvn sonar:sonar -Dsonar.host.url=$SONARQUBE_URL:$SONARQUBE_PORT  -Dsonar.login=fd8bfb60348c7b145dd631dc4ff1d88990ddebbe"
       // sh " mvn sonar:sonar  -Dsonar.host.url=http://sonarqube:9000  -Dsonar.login=fd8bfb60348c7b145dd631dc4ff1d88990ddebbe" 
	     println "done."
		

     }
    }
   }
   post {
    always {
     // using warning next gen plugin
     recordIssues aggregatingResults: true, tools: [mavenConsole(), java(), javaDoc(), checkStyle(pattern: '**/target/checkstyle-result.xml'), findBugs(pattern: '**/target/findbugsXml.xml', useRankAsPriority: true), pmdParser(pattern: '**/target/pmd.xml')]
    }
   }
  }
  stage('Deploy Artifact To Nexus') {
   when {
    anyOf { branch 'master'; branch 'develop'  ; branch 'main'}
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
   when {
    anyOf { branch 'master'; branch 'develop'  ; branch 'main'}
   }
   agent {
    docker {
     image 'ahmed24khaled/ansible-management'
     reuseNode true
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
   when {
	   anyOf { branch 'master'; branch 'main'}
   }
   agent {
    docker {
     image 'ahmed24khaled/ansible-management'
     reuseNode true
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
}
