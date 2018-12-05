node {
    
   /* stage('preparation')  {
         def userInput = input(
        id: 'userInput', message: 'preparation',
                        parameters: [
                                    [$class: 'TextParameterDefinition', defaultValue: 'cicd', description: 'dev env', name: 'DEV_PROJECT'],
                                   [$class: 'TextParameterDefinition', defaultValue: 'test1', description: 'stage env', name: 'STAGE_PROJECT']
                                 ] )
                      devenv = userInput.DEV_PROJECT
                      stageenv = userInput.STAGE_PROJECT
                      input message: "Is the PARAMETERS are correctly set for deployment?", ok: "Promote" 
        
    }*/
      def mvnHome
      mvnHome = tool 'maven'
      stage('Fetch Code') { 
        	git changelog: false, credentialsId: 'b7552763e472555d9593ff49af979d3e7a879884', url: 'https://github.com/sudhirkr448/boxfuse-sample-java-war-hello.git'
   }
      stage('Build') {
       	echo "in Build"
       sh "'${mvnHome}/bin/mvn' -Dmaven.test.failure.ignore clean install"
   }    
      stage('Sonar') {
       echo "in sonar" 
   	sh "'${mvnHome}/bin/mvn' -X sonar:sonar -Dsonar.host.url=http://10.131.0.202:9000 -Dsonar.login=52d9a722dfdb899d50688842eb9b77f27cf5ff47 '-fpom.xml' '-Dsonar.projectKey=test' '-Dsonar.language=java' '-Dsonar.sources=src'"
   }
   stage('Nexus'){
       	echo "in Nexus"
   	sh "curl -v -u admin:admin123 --upload-file /var/lib/jenkins/jobs/cicd/jobs/cicd-tasks-pipeline/workspace/target/hello-1.0.war  http://10.131.0.201:8081/repository/myapp/"
   }
   stage('Build Image') {
       	openshift.withCluster() {
       		openshift.withProject(env.DEV_PROJECT) {
     	       		    def bcSelector = openshift.selector("bc", "jboss")
                        def bcExists = bcSelector.exists()
                   if (!bcExists) { 
       		    	openshift.newBuild("--name=jboss", "--image-stream=jboss-eap70-openshift:1.5", "--binary=true") 
                   } else {
                       echo "The specified image already exists"
                   }
       		    	
       		}}
       		    
   }    
      stage('Build Image with app') {
        sh "rm -rf oc-build && mkdir -p oc-build/deployments"
        sh "cp /var/lib/jenkins/jobs/cicd/jobs/cicd-tasks-pipeline/workspace/target/hello-1.0.war oc-build/deployments/ROOT.war"                                
           openshift.withCluster() {
             openshift.withProject(env.DEV_PROJECT) {
               openshift.selector("bc", "jboss").startBuild("--from-dir=oc-build", "--wait=true")
             }
           }
      }
      stage('deploy to Dev') {
                              openshift.withCluster() {
          openshift.withProject(env.DEV_PROJECT) {
            if (openshift.selector('dc', 'jboss').exists()) {
              openshift.selector('dc', 'jboss').delete()
              openshift.selector('svc', 'jboss').delete()
              openshift.selector('route', 'jboss').delete()
            }

            def app = openshift.newApp("jboss:latest")
            app.narrow("svc").expose();
            def dc = openshift.selector("dc", "jboss")
         //openshiftScale(namespace: 'cicd', deploymentConfig: 'jboss',replicaCount: '2')
         openshift.tag("${env.DEV_PROJECT}/jboss:latest", "${env.DEV_PROJECT}/jboss:${build_number}")
         }
   	}
   }
   stage('Promote to TEST?') {
         timeout(time:15, unit:'MINUTES') {
         input message: "Promote to test?", ok: "Promote"
         }
       openshift.withCluster() {
         def version = "test"
         openshift.tag("${env.DEV_PROJECT}/jboss:${build_number}", "${env.STAGE_PROJECT}/jboss:${version}")
         openshift.withProject(env.STAGE_PROJECT) {
            if (openshift.selector('dc', 'jboss').exists()) {
            openshift.selector('dc', 'jboss').delete()
            openshift.selector('svc', 'jboss').delete()
            openshift.selector('route', 'jboss').delete()
            }
          openshift.newApp("jboss:${version}").narrow("svc").expose()
          //openshiftScale(namespace: 'test', deploymentConfig: 'jboss',replicaCount: '1')
         }
   	}
   }
}
