def label = "worker-${UUID.randomUUID().toString()}"

podTemplate(label: label, cloud: "academycluster", containers: [
  containerTemplate(name: 'maven', image: 'maven:3.6.2-jdk-8', command: 'cat', ttyEnabled: true),
  containerTemplate(name: "scanner", image: "newtmitch/sonar-scanner", ttyEnabled: true, command: "cat"),
  containerTemplate(name: 'docker', image: 'docker:dind', command: 'dockerd --host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2375 --storage-driver=overlay', ttyEnabled: true, alwaysPullImage: true, privileged: true),
  containerTemplate(name: 'kubectl', image: 'roffe/kubectl', command: 'cat', ttyEnabled: true)
]) {
  node(label) {

	try {

		stage('scm') {
			git credentialsId: 'post-academy-git-auth', url: 'https://github.com/maxHyeon/cicd-backend.git', branch: 'master'
	    }

    	def props = readProperties  file:"./kubernetes/pipeline.properties"
    	def tag = props["version"]
    	def dockerRegistry = props["dockerRegistry"]
    	def credentialRegistry=props["credentialRegistry"]
    	def image = props["image"]
    	def appname = props["appname"]
    	def containername = props["containerName"]
    	def deployment = props["deployment"]
    	def service = props["service"]
    	def namespace = props["namespace"]
    	def sonarQubeURL = props["sonarQubeURL"]
    	def sonarQubeProject = props["sonarQubeProject"]

	    def timeStamp = System.currentTimeMillis()
	    echo "TimeStamp: ${timeStamp}"
	    tag = tag+"-"+timeStamp
		

		stage('Build') {
			container('maven') {
				sh "mvn -B -DskipTests clean package"
			}
		}

		stage('Static Code Analysis') {
			container('scanner') {
				sh "sonar-scanner \
					-Dsonar.projectKey=${sonarQubeProject} \
				  -Dsonar.projectBaseDir=./\
					-Dsonar.java.binaries=target/classes\
					-Dsonar.host.url=${sonarQubeURL} \
					-Dsonar.login=60464e819894dc41b45f3eca168784c1b7d76e55"

			}
		}

		stage('Dockerizing') {
			container('docker') {
				docker.withRegistry("${dockerRegistry}", "${credentialRegistry}") {
					sh "docker build -f ./Dockerfile -t ${image}:${tag} ${WORKSPACE}"
					sh "docker push ${image}:${tag}"
					sh "docker tag ${image}:${tag} ${image}:latest"
					sh "docker push ${image}:latest"
				}
			}
		}
        
		stage('Deploy') {
      		container('kubectl') {
				sh "kubectl apply -n ${namespace} -f ${deployment}"
				sh "kubectl apply -n ${namespace} -f ${service}"
				sh "kubectl set image deployment ${appname} ${containername}=${image}:${tag} -n ${namespace}"
				sh "kubectl get svc -n lab23"
      		}
    	}

	} catch(e) {

	}
  }
}