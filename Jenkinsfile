//AP
def buildVersion = null
properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '5']]])
stage 'Build'
if(!env.JOB_NAME.startsWith("beedemo-api/mobile-deposit-api/")) {
  echo 'only build for beedemo-api GitHub org folder'
  error 'invalid project path'
}
node('docker-cloud') {
    checkout scm
    sh('git rev-parse HEAD > GIT_COMMIT')
    git_commit=readFile('GIT_COMMIT')
    docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
        sh "mvn -Dmaven.repo.local=/data/mvn/repo -DGIT_COMMIT='${git_commit}' -DBUILD_NUMBER=${env.BUILD_NUMBER} -DBUILD_URL=${env.BUILD_URL} clean package"
    }
    stash name: 'pom', includes: 'pom.xml'
    stash name: 'jar-dockerfile', includes: '**/target/*.jar,**/target/Dockerfile'
}

if(!env.BRANCH_NAME.startsWith("PR")){
  checkpoint 'Build Complete'
  stage 'Quality Analysis'
  node('docker-cloud') {
    try {
    unstash 'pom'
    //test in paralell
    parallel(
        integrationTests: {
            docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
                sh 'mvn -Dmaven.repo.local=/data/mvn/repo verify'
            }
        }, sonarAnalysis: {
            withCredentials([[$class: 'StringBinding', credentialsId: 'sonar.beedemo', variable: 'TOKEN']]) {
                echo 'running sonar tests'
                docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
                    sh 'mvn -Dmaven.repo.local=/data/mvn/repo -Dsonar.scm.disabled=True -Dsonar.login=$TOKEN sonar:sonar'
                }
                echo 'finished sonar tests'
            }
        }, failFast: true
    )
    } catch (x) {
      currentBuild.result = "failed"
      hipchatSend color: 'RED', message: "${env.JOB_NAME} ${env.BUILD_NUMBER} status: ${currentBuild.result} <a href=\'${env.BUILD_URL}\'>Open</a>", room: '1613593', server: 'cloudbees.hipchat.com', token: 'A6YX8LxNc4wuNiWUn6qHacfO1bBSGXQ6E1lELi1z', v2enabled: true
      mail body: "Job '${env.JOB_NAME}' has failed.  See <a href=\"${env.BUILD_URL}\">logs</a> for details.", mimeType: 'text/html', subject: "${env.JOB_NAME} FAILURE", to: 'kmadel@cloudbees.com'
      throw x
    }
  }
}

if(env.BRANCH_NAME=="master"){
  checkpoint 'Quality Analysis Complete'
  stage name: 'Version Release', concurrency: 1
  node('docker-cloud') {
    unstash 'pom'

    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    if (matcher) {
        buildVersion = matcher[0][1]
        echo "Release version: ${buildVersion}"
    }
    matcher = null
    
    docker.withServer('tcp://52.27.249.236:3376', 'beedemo-swarm-cert'){

        stage 'Build Docker Image'
        def mobileDepositApiImage
        //unstash Spring Boot JAR and Dockerfile
        unstash 'jar-dockerfile'
        dir('target') {
            mobileDepositApiImage = docker.build "kmadel/mobile-deposit-api:${buildVersion}"
        }
        
        stage 'Publish Docker Image'
        sh "docker -v"
        //use withDockerRegistry to make sure we are logged in to docker hub registry
        withDockerRegistry(registry: [credentialsId: 'docker-registry-kmadel-login']) { 
          mobileDepositApiImage.push()
        }

        stage 'Deploy to Prod'
        try{
          sh "docker stop beedemo-swarm-master/mobile-deposit-api"
          sh "docker rm beedemo-swarm-master/mobile-deposit-api"
        } catch (Exception _) {
           echo "no container to stop"        
        }
        //docker traceability rest call
        container = mobileDepositApiImage.run("--name mobile-deposit-api -p 8080:8080 --env='constraint:node==beedemo-swarm-master'")
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'webhook_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME']]) {
          sh "curl http://${env.USERNAME}:${env.PASSWORD}@api-team.jenkins-latest.beedemo.net/docker-traceability/submitContainerStatus \
             --data-urlencode status=deployed \
             --data-urlencode hostName=prod-server-1 \
             --data-urlencode hostName=prod \
             --data-urlencode imageName=cloudbees/mobile-deposit-api \
             --data-urlencode inspectData=\"\$(docker inspect ${container.id})\""
        }
     }
  }
}
node('docker-cloud') {
  //update hipchat with success
  currentBuild.result = "success"
  hipchatSend color: 'GREEN', message: "${env.JOB_NAME} ${env.BUILD_NUMBER} status: ${currentBuild.result} <a href=\'${env.BUILD_URL}\'>Open</a>", room: '1613593', server: 'cloudbees.hipchat.com', token: 'A6YX8LxNc4wuNiWUn6qHacfO1bBSGXQ6E1lELi1z', v2enabled: true
}
