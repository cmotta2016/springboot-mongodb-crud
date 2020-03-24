timestamps {
    node('maven') {
        stage('Checkout'){
            checkout scm
            //checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/cmotta2016/springboot-mongodb-crud.git']]])
        }
        stage('Compile with Azure Artifacts'){
	        sh 'mvn -gs /home/jenkins/.m2/settings.xml clean install'
        }
        stage('Build with S2I'){
            sh 's2i build . cmotta2016/java-8-bases2i:latest azureterraformb512626c.azurecr.io/java-mongo:${BUILD_NUMBER} --loglevel 5 --assemble-user root --network host --volume /home/jenkins/repository:/tmp/artifacts/m2 --inject /opt/artifacts:/opt/jboss/container/maven/default'
        }
        stage('Push Image to ACR'){
            withCredentials([usernamePassword(credentialsId: 'acr-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
            sh '''
            docker login -u "$DOCKER_USERNAME" -p "$DOCKER_PASSWORD" azureterraformb512626c.azurecr.io
            docker tag azureterraformb512626c.azurecr.io/java-mongo:${BUILD_NUMBER} azureterraformb512626c.azurecr.io/java-mongo:latest
            docker push azureterraformb512626c.azurecr.io/java-mongo:${BUILD_NUMBER}
            docker push azureterraformb512626c.azurecr.io/java-mongo:latest
            docker rmi -f azureterraformb512626c.azurecr.io/java-mongo:${BUILD_NUMBER} azureterraformb512626c.azurecr.io/java-mongo:latest
            '''
            }
        }
        stage('Deploy QA'){
            sh 'kubectl config set-context java-mongodb --namespace=java-mongodb && kubectl config use-context java-mongodb'
            def deployment = sh(script: "kubectl get deployment java-mongo -o jsonpath='{ .metadata.name }' --ignore-not-found", returnStdout: true).trim()
            if (deployment == "java-mongo") {
                sh 'kubectl set image deployment/java-mongo java-mongo=azureterraformb512626c.azurecr.io/java-mongo:${BUILD_NUMBER} --record'
            }
            else {
                sh 'kubectl create -f deployment.yaml --validate=false'
                sh 'kubectl set image deployment/java-mongo java-mongo=azureterraformb512626c.azurecr.io/java-mongo:${BUILD_NUMBER} --record'
            }
            sh 'kubectl rollout status deployment.apps/java-mongo'
        }
        stage('Test Deployment'){
            routeHost = sh(script: "kubectl get ingress java-mongo -o jsonpath='{ .spec.rules[0].host }'", returnStdout: true).trim()
            echo "${routeHost}"
        }
    }
}
