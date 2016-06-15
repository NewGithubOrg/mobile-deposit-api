def buildVersion = null
properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '5']]])
stage 'Build'
node('docker-cloud') {
    checkout scm
    docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
        sh 'mvn -Dmaven.repo.local=/data/mvn/repo -Dsonar.jdbc.username=NULL -Dsonar.jdbc.password=NULL clean package'
    }
    stash name: 'pom', includes: 'pom.xml, src, target'
}

if(!env.BRANCH_NAME.startsWith("PR")){
  checkpoint 'Build Complete'
  stage 'Quality Analysis'
  node('docker-cloud') {
    unstash 'pom'
    //test in paralell
    parallel(
        integrationTests: {
            docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
                //sh 'mvn -Dmaven.repo.local=/data/mvn/repo -Dsonar.jdbc.username=NULL -Dsonar.jdbc.password=NULL verify'
                echo 'run integration tests'
            }
        }, sonarAnalysis: {
           // withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'sonar.beedemo',
                //usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                echo 'run sonar tests'
                //need to fix sonarAnalysis
                //docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
                    //sh 'mvn -Dmaven.repo.local=/data/mvn/repo -Dsonar.scm.disabled=True -Dsonar.jdbc.username=$USERNAME -Dsonar.jdbc.password=$PASSWORD sonar:sonar'
                //}
            }
        }, failFast: true
    )
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
        dir('target') {
            mobileDepositApiImage = docker.build "kmadel/mobile-deposit-api:${buildVersion}"
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
        sh "curl http://webhook:018ebf0660a74b561b852105e35a33b6@jenkins-latest.beedemo.net/api-team/docker-traceability/submitContainerStatus \
            --data-urlencode status=deployed \
            --data-urlencode hostName=prod-server-1 \
            --data-urlencode hostName=prod \
            --data-urlencode imageName=cloudbees/mobile-deposit-api \
            --data-urlencode inspectData=\"\$(docker inspect $container.id)\""
        
        
        stage 'Publish Docker Image'
        sh "docker -v"
        //use withDockerRegistry to make sure we are logged in to docker hub registry
        withDockerRegistry(registry: [credentialsId: 'docker-registry-kmadel-login']) { 
          mobileDepositApiImage.push()
        }
     }
  }
}
