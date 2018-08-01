pipeline {
    agent {
        node {
            label 'docker-io-ui'
        }
    }
    environment {
        jenkins_secret_username="ae029532-9015-4bd0-b7f0-d9ee65598885"
        jenkins_secret_password="5db8122c-c284-4933-b879-3391bef71e0b"
    }

    options { skipDefaultCheckout() }

    stages {
        stage ('Initialize Application Migration') {
            agent none
            steps {
                script {
                        withCredentials(
                                        [string(credentialsId: jenkins_secret_username, variable: 'SECRET_ID'),
                                        string(credentialsId: jenkins_secret_password, variable: 'SECRET_PASS')]) 
                        {
                                stage ('Checkout') {
                                        echo 'Checking out SCM'
                                        checkout scm

                                        sh '''
                                                echo "PATH = ${PATH}"
                                                echo "M2_HOME = ${M2_HOME}"
                                        '''
                                }

                                stage ('Java Build') {
                                        echo 'Maven Build'
                                        sh 'cd webapp && mvn clean install'
                                }

                                stage ('Java Tests') {
                                        echo 'Place holder to run Unit tests.'
                                        //sh '%%%UnitTestParam$$$'
                                        sh 'sleep 3'

                                }

                                stage ('SonarQube Analysis') {
                                        def gitUrl = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
                                        echo " Performing Sonar Analysis"
                                        sh 'sleep 10'
                                }

                                stage ('Docker Build') {
                                        retry(3) {
                                                sh 'docker login -u $SECRET_ID -p $SECRET_PASS cloudin.dst.ibm.com:8500'
                                        }
                                        echo 'Building image.'
                                        sh 'cp webapp/target/everest-1.0-SNAPSHOT.war . && docker build --no-cache -t cloudin.dst.ibm.com:8500/nextgen-tool-ns/sample-jboss:latest . && rm -f everest-1.0-SNAPSHOT.war'
                                        echo 'Pushing image taggeed :${env.BUILD_ID}.'
                                        retry(3) {
                                                sh 'docker push cloudin.dst.ibm.com:8500/nextgen-tool-ns/sample-jboss:latest'
                                        }
                                        echo 'Pushing image taggeed :latest.'
                                }

                                stage ('Kubernetes deploy') {
                                        echo 'Setting build No. for k8s deployment.'
                                        echo 'Deploying service.'
                                        sh 'kubectl config set-cluster cluster.local --server=https://9.193.199.44:8001 --insecure-skip-tls-verify=true'

                                        sh 'kubectl config set-context cluster.local-context --cluster=cluster.local'

                                        sh 'export icp_username=$SECRET_ID; export icp_password=$SECRET_PASS; section=$(curl -k -X POST -H "Content-Type: application/x-www-form-urlencoded;charset=UTF-8" -d "grant_type=password&username=$icp_username&password=$icp_password&scope=openid" https://cloudin.dst.ibm.com:8443/idprovider/v1/auth/identitytoken| sed \'s/[{}"]//g\');arr=($(echo $section | tr "," "\n")); icp_token=$(for i in ${arr[@]}; do echo $i; done| sed -n "/id_token/s/id_token://p");kubectl config set-credentials ntooling@in.ibm.com --token=$icp_token'

                                        sh 'kubectl config set-context cluster.local-context --user=ntooling@in.ibm.com --namespace=nextgen-tool-ns'
                                        sh 'kubectl config use-context cluster.local-context'

                                        sh 'kubectl delete --ignore-not-found=true -f deployment.yaml'
                                        sh "kubectl create -f deployment.yaml "
                                        
                                        sh "kubectl delete --ignore-not-found=true -f service.yaml "
                                        sh "kubectl create -f service.yaml "
                                }
                        }
                }
            }
        }
    }
}
