#!groovy
// Jenkinsfile

def DOCKER_HUB_USER="etejeda"
def CONTAINER_NAME="dockerxc"
def CONTAINER_TAG="beta${BUILD_NUMBER}"
def HTTP_PORT="8089"
def HTTP_PORT_PROD="80"
def NAMEJOB="TEST-UTALKS2019"
def WORKDIRLOCAL="/var/lib/docker/volumes/jenkins-data/_data/workspace/"
// Test
pipeline {

        agent { node { label 'master' } }

        environment {
            CI = 'false'
        }
        stages {
            stage('Git Checkout'){
                steps {
                     checkout scm
                }
            }
            stage('Prepare Images') {
                steps {
                    echo 'Run Docker Apache...'
                    sh "docker run -d --rm -p ${HTTP_PORT}:80 --name httpd-apache2 -v ${WORKDIRLOCAL}${NAMEJOB}/WWW:/usr/local/apache2/htdocs/ httpd:2.4"
                    echo 'Run Selenium Hub in the network grid...'
                    sh 'docker run --rm -d -p 4444:4444 --net grid --name selenium-hub-testing-1 selenium/hub'
                    echo 'Run a node of firefox... (Selenium Grid)'
                    sh 'docker run --rm -d --net grid --name node-firefox-1 -e HUB_HOST=selenium-hub-testing-1 -e START_XVFB=false -v /dev/shm:/dev/shm selenium/node-firefox'
                }
            }
            stage('Build') {
                steps {
                    echo 'Start Install Dependencies...'
                    sh "docker run --rm --name npm-pipeline -v ${WORKDIRLOCAL}${NAMEJOB}/Testing:/usr/src/app -w /usr/src/app node:8-alpine npm install"
                }
            }
            stage('Test') {
                steps {
                    echo 'Start Testing (Mocha)...'
                    sh "docker run --rm --name npm-pipeline -v ${WORKDIRLOCAL}${NAMEJOB}/Testing:/usr/src/app -w /usr/src/app node:8-alpine npm test"

                }
            }
            stage('Deploy') {
                steps {
                    imageBuild(CONTAINER_NAME, CONTAINER_TAG)
                withCredentials([usernamePassword(credentialsId: 'dockerHubAccount', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                           pushToImage(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD)
                  }

                    runApp(CONTAINER_NAME, CONTAINER_TAG, DOCKER_HUB_USER, HTTP_PORT_PROD)
                }
            }


        }
        post {
            always {
                echo '===== ALWAYS ====='
                imagePrune(CONTAINER_NAME)
            }
        }

}




def imageBuild(containerName, tag){
    sh "docker build -t $containerName:$tag  -t $containerName --pull --no-cache ."
    echo "Image build complete"
}

def pushToImage(containerName, tag, dockerUser, dockerPassword){
    sh "docker login -u $dockerUser -p $dockerPassword"
    sh "docker tag $containerName:$tag $dockerUser/$containerName:$tag"
    sh "docker push $dockerUser/$containerName:$tag"
    echo "Image push complete"
}

def imagePrune(containerName){
    try {
        print 'Stop all services..'

        sh "docker stop node-firefox-1"
        sh "docker stop selenium-hub-testing-1"
        sh "docker stop httpd-apache2"

    } catch(error){}
}

def runApp(containerName, tag, dockerHubUser, httpPort){
    sh "docker pull $dockerHubUser/$containerName:$tag"
    try {
        print 'Stop all services..'
        sh "docker stop $containerName"
        sh "docker rm $containerName"


    } catch(error){
    print 'Error Stop Container Prod'
    }finally{
      sh "docker run -d --rm -p $httpPort:80 --name $containerName  $dockerHubUser/$containerName:$tag"
      echo "Application started on port: $httpPort (http)"
    }


}
