node {
    def mvnHome
    mvnHome = tool 'maven'
    stage('Fetch Code') { 
         git changelog: false, credentialsId: 'b7552763e472555d9593ff49af979d3e7a879884', poll: false, url: 'http://10.130.3.94:3000/gogs/cicd-demo.git'
    }
    stage('Build') {
         echo "in Build"
         sh "'${mvnHome}/bin/mvn' -Dmaven.test.failure.ignore clean install"
    }
    
    stage('Sonar') {
       echo "in sonar"
      sh "'${mvnHome}/bin/mvn' -X sonar:sonar -Dsonar.host.url=http://10.131.4.248:9000  -Dsonar.login=73c67f19dff039d46fc157c87e413e45f9c9db9e '-fpom.xml' '-Dsonar.projectKey=cicd' '-Dsonar.language=java' '-Dsonar.sources=src'"
   }
     stage('Nexus'){
       echo "in Nexus"
       sh "curl -v -u admin:admin123 --upload-file /var/lib/jenkins/jobs/devpipeline/jobs/devpipeline-tasks-pipeline/workspace/target/hello-1.0.war http://10.130.3.98:8081/repository/myapp/"
   }
     stage('Build Image') {
           openshift.withCluster() {
                      openshift.withProject(env.DEV_PROJECT) {
                        //return !openshift.selector("bc", "jboss").exists();
                        openshift.newBuild("--name=jboss", "--image-stream=jboss-eap70-openshift:1.5", "--binary=true")
                      }
                    }}
    
    stage('Build Image with app') {
       sh "rm -rf oc-build && mkdir -p oc-build/deployments"
        sh "cp /var/lib/jenkins/jobs/devpipeline/jobs/devpipeline-tasks-pipeline/workspace/hello-1.0.war oc-build/deployments/ROOT.war"
                        openshift.withCluster() {
                      openshift.withProject(env.DEV_PROJECT) {
                        openshift.selector("bc", "jboss").startBuild("--from-dir=oc-build", "--wait=true")
                        openshift.tag("${env.DEV_PROJECT}/jboss:latest", "${env.DEV_PROJECT}/jboss:devpipeline")
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
                    def app = openshift.newApp("jboss:devpipeline")
                        app.narrow("svc").expose();
                        def dc = openshift.selector("dc", "jboss")
                        while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                            sleep 10
                        }
                        openshift.set("triggers", "dc/jboss", "--manual")
                        openshiftScale(namespace: 'devpipeline', deploymentConfig: 'jboss',replicaCount: '3')

                              }}}
				stage('Promote to TEST?') {
                  timeout(time:15, unit:'MINUTES') {
                      input message: "Promote to test?", ok: "Promote"
                  }
                     openshift.withCluster() {
                        def version = "testing"
                      openshift.tag("${env.DEV_PROJECT}/jboss:devpipeline", "${env.STAGE_PROJECT}/jboss:${version}")
                       openshift.withProject(env.STAGE_PROJECT) {
                        if (openshift.selector('dc', 'jboss').exists()) {
                          openshift.selector('dc', 'jboss').delete()
                          openshift.selector('svc', 'jboss').delete()
                          openshift.selector('route', 'jboss').delete()
                        }
                        openshift.newApp("jboss:${version}").narrow("svc").expose()
                        openshiftScale(namespace: 'test', deploymentConfig: 'jboss',replicaCount: '3')
                  
                }}}
}
