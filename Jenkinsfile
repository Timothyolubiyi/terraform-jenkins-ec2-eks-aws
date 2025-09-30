pipeline {


    agent any
    stages {
        
        stage("GitHub checkout....") {
            steps {
                script {
 
                    git branch: 'main', url: 'https://github.com/Timothyolubiyi/Python-flask.git' 
                }
            }
        }
        stage("Build docker connecting....."){
            steps{
                sh 'printenv'
                sh 'git version'
                sh 'docker build . -t smartgigsctf/f-app3.1'
            }
        }

        stage("push image to DockerHub"){

            steps{


                script {

                
                  
                 withCredentials([string(credentialsId: 'ACCESSID', variable: 'ACCESSID')]) {

                    sh 'docker login -u smartgigsctf -p ${ACCESSID}'
                }
                    sh 'docker push smartgigsctf/f-app3.1:latest'
                }
            }
        }
        stage('Deploying python app to Kubernetes') {


            steps {

                script {

                    dir('kubernetes') {

                        sh ('aws eks update-kubeconfig --name eks-cluster-200 --region eu-north-1')
                        sh 'kubectl config current-context'
                        sh "kubectl get ns"
                        sh "kubectl apply -f deployment.yaml"
                        sh "kubectl apply -f service.yaml"
                    }
      

                }
            }
        }
        stage('AWS') {
      steps {
        // This requires the "AWS Steps" / "AWS Credentials" binding support in Jenkins (plugin)
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            aws sts get-caller-identity
            # other aws commands...
          '''
        }
      }
    }
  }
}