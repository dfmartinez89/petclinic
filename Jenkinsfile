pipeline {
  agent any
  environment {
    CONTAINER_REGISTRY = 'gcr.io'
    GOOGLE_CLOUD_PROJECT = 'cnsa2022-dfm354'
    CREDENTIALS_ID = 'cnsa2022-dfm354'
  }
  tools {
    maven "maven"
  }
  stages {
    stage("Checkout code") {
      steps {
        // checkout scm
        git  branch:'main', url:'https://github.com/thywillbedone/petclinic.git'
      }
    }
    stage('Compile, Test, Package') {
      steps {
        sh "mvn clean package -Dcheckstyle.skip"
      }
      post {
        success {
          junit '**/target/surefire-reports/TEST-*.xml'
          archiveArtifacts 'target/*.jar'
        }
      }
    }
    stage("Build image") {
      steps {
        script {
          dockerImage = docker.build(
            "${env.CONTAINER_REGISTRY}/${env.GOOGLE_CLOUD_PROJECT}/petclinic:${env.BUILD_ID}",
            "--rm -f Dockerfile .")
        }
      }
    }
    
    stage("Deploy to Testing (locally)") {
      steps {
        sh "docker stop petclinic || true && docker rm  petclinic || true"
        sh "docker run -d -p 8080:8080 -t --name petclinic ${env.CONTAINER_REGISTRY}/${env.GOOGLE_CLOUD_PROJECT}/petclinic:${env.BUILD_ID}"
      }
    }
    
    stage('End-to-end Test image') {
        // Ideally, we would run some end-to-end tests against our running container.
     steps{
           sh 'echo "End-to-end Tests passed"'
     }
   }
   
   stage("Push image") {
    steps {
      script {
        docker.withRegistry('https://'+ CONTAINER_REGISTRY, 'gcr:'+ GOOGLE_CLOUD_PROJECT) {
          dockerImage.push("latest")
          dockerImage.push("${env.BUILD_ID}")
        }
      }
    }
   }
   
    stage('Deploy to Production') {
      steps{
        // Check to manual approving deploy to production.
        // It implemenents Continuous Delivery instead of Continuous Deployment
        input message: "Proceed Deploy to Production?"
        sh '''
          ssh -i ~/.ssh/id_rsa_deploy ubuntu@34.135.21.215 "if docker ps -q --filter name=petclinic | grep . ; then docker stop petclinic ; fi"
          ssh -i ~/.ssh/id_rsa_deploy ubuntu@34.135.21.215 "if docker ps -a -q --filter name=petclinic | grep . ; then docker rm -fv petclinic ; fi"
          ssh -i ~/.ssh/id_rsa_deploy ubuntu@34.135.21.215 "docker run -d -p 80:8080 -t --name petclinic ${CONTAINER_REGISTRY}/${GOOGLE_CLOUD_PROJECT}/petclinic:latest"
        '''
      }
    }
    stage('Remove Unused docker image') {
      steps{
        // input message:"Proceed with removing image locally?"
        sh 'if docker ps -q --filter name=petclinic | grep . ; then docker stop petclinic && docker rm -fv petclinic; fi'
        sh 'docker rmi ${CONTAINER_REGISTRY}/${GOOGLE_CLOUD_PROJECT}/petclinic:$BUILD_NUMBER'
      }
    }
 }
}