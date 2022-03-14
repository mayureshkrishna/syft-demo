pipeline {
  
  environment {
    // shouldn't need the registry variable unless you're not using dockerhub
    // registry = 'registry.hub.docker.com'
    //
    // change this HUB_CREDENTIAL to the ID of whatever jenkins credential has your registry user/pass
    // first let's set the docker hub credential and extract user/pass
    // we'll use the USR part for figuring out where are repository is
    HUB_CREDENTIAL = "docker-hub"
    // use credentials to set DOCKER_HUB_USR and DOCKER_HUB_PSW
    DOCKER_HUB = credentials("${HUB_CREDENTIAL}")
    // change repository to your DockerID
    REPOSITORY = "${DOCKER_HUB_USR}/${JOB_BASE_NAME}"
    // what package do we want to block? (this is optional, see the "analyze with syft" stage)
    // BLOCKED_PACKAGE = "curl"
  } // end environment
  
    agent {
    kubernetes {
      label 'spring-petclinic-demo'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
labels:
  component: ci
spec:
  # Use service account that can deploy to all namespaces
  serviceAccountName: jenkins
  containers:
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
"""
}
   }
  stages {
    
    stage('Checkout SCM') {
      steps {
        checkout scm
      } // end steps
    } // end stage "checkout scm"
   
    stage('Verify Tools') {
      steps {
        // check for docker and curl,
        // install/update syft, /var/jenkins_home should be writable 
        // also if you've set up jenkins in a docker container, this dir should be a persistent volume
        sh """
          which curl
          curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /home/jenkins/agent/
          """
      } // end steps
    } // end stage "Verify Tools"
    
    stage('Build Image') {
      steps {
        sh """
          docker login -u ${DOCKER_HUB_USR} -p ${DOCKER_HUB_PSW}
          docker build -t ${REPOSITORY}:${BUILD_NUMBER} --pull -f ./Dockerfile .
        """
      } // end steps
    } // end stage "build and push"
    
    // I don't like using the docker plugin, but if you do:
   //  stage('Build image and tag with build number') {
   //   steps {
   //     script {
   //       dockerImage = docker.build REPOSITORY + ":${BUILD_NUMBER}"
   //     } // end script
  //     } // end steps
  //   } // end stage "build image and tag w build number"
    
    stage('Analyze with syft') {
      steps {
        // run syft and output to file, we'll archive that at the end
        sh '/home/jenkins/agent/syft -o cyclonedx-json ${REPOSITORY}:${BUILD_NUMBER} > ${JOB_BASE_NAME}.cyclonedx.json'
        //
        // you can do some analysis here, for example you can check for
        // forbidden packages and break the pipeline if the image has
        // (e.g.) curl, sudo, or something else dangerous installed.
        // There is a variable in the environment section at the top of this
        // Jenkinsfile, you can uncomment that and set it to whatever and then use this:
        //
        // sh """
        //   /var/jenkins_home/bin/syft -o spdx-json ${REPOSITORY}:${BUILD_NUMBER} | \
        //   tee ${JOB_BASE_NAME}.spdx.json | \
        //   jq .packages[].name | \
        //   tr "\n" " " | grep -qv ${BLOCKED_PACKAGE}
        // """
        //
        // this method uses native syft sbom instead of spdx:
        // sh '/var/jenkins_home/bin/syft -o json ${REPOSITORY}:${BUILD_NUMBER} | jq .artifacts[].name | tr "\n" " " | grep -qv curl'
        //
      } // end steps
    } // end stage "analyze with syft"
    
    stage('Promote and Push Image') {
      steps {
        sh """
          docker tag ${REPOSITORY}:${BUILD_NUMBER} ${REPOSITORY}:prod
          docker push ${REPOSITORY}:prod
        """
        // I don't really like using the docker plug-in, but if you do, something like this:
     //   script {
     //     docker.withRegistry('', HUB_CREDENTIAL) {
     //       dockerImage.push('prod') 
        //    // dockerImage.push takes the argument as a new tag for the image before pushing
          }
        } // end script
      } // end steps
    } // end stage "retag as prod"
    
  } // end stages
  
  post {
    always {
      // archive the sbom
      archiveArtifacts artifacts: '*.cyclonedx.json'
      // delete the images locally
     // sh 'docker image rm ${REPOSITORY}:${BUILD_NUMBER} ${REPOSITORY}:prod || failure=1'
    } // end always
  } //end post
      
} //end pipeline
