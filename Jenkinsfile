pipeline {
    agent any
    
    parameters{

        choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
        
    }
    tools { 
        maven 'maven-3.8.6' 
    }
    environment{

        ACCESS_KEY = credentials('AWS_ACCESS_KEY_ID')
        SECRET_KEY = credentials('AWS_SECRET_KEY_ID')
    }

    stages {
        stage('Checkout git') {
            steps {
              git 'https://github.com/praveensirvi1212/medicure-project.git'
            }
        }
        
        stage ('Build & JUnit Test') {
            steps {
                sh 'mvn clean package' 
            }
        }
        
        stage ('Docker Build and push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', passwordVariable: 'password', usernameVariable: 'username')]) {
                    
                    sh 'docker build -t praveensirvi/medicure:v1 .'
                    sh "docker login -u ${username} -p ${password} "
                    sh 'docker push praveensirvi/medicure:v1'
                    
                    
                }
            }
        }
        stage('Create EKS Cluster : Terraform'){
            when { expression {  params.action == 'create' } }
            steps{
                script{

                    dir('eks_module') {
                      sh """
                          
                          terraform init 
                          terraform plan -var 'access_key=$ACCESS_KEY' -var 'secret_key=$SECRET_KEY'  --var-file=./config/terraform.tfvars
                          terraform apply -var 'access_key=$ACCESS_KEY' -var 'secret_key=$SECRET_KEY'  --var-file=./config/terraform.tfvars --auto-approve
                      """
                  }
                }
            }
        }
        
        stage('Connect to EKS '){
            when { expression {  params.action == 'create' } }
        steps{

            script{

                sh """
                aws configure set aws_access_key_id "$ACCESS_KEY"
                aws configure set aws_secret_access_key "$SECRET_KEY"
                aws configure set region ap-south-1
                aws eks --region ap-south-1 update-kubeconfig --name EKS-cluster
                """
            }
        }
        }
        
        stage('Deployment on test-EKS Cluster'){
            when { expression {  params.action == 'create' } }
            steps{
                script{
                  
                  def apply = false

                  try{
                    input message: 'please confirm to deploy on test-eks cluster', ok: 'Ready to apply the config ?'
                    apply = true
                  }catch(err){
                    apply= false
                    currentBuild.result  = 'UNSTABLE'
                  }
                  if(apply){

                    sh """
                      kubectl apply -f test-cluster-deployment.yaml
                    """
                  }
                }
            }
        } 
        
        
        stage ("wait_for_application to come up"){
            steps {
              sh 'sleep 40'
            }
        }
        
        stage('Selenium test cases') {
            steps {
              sh 'java -jar Selenium.jar'
            }
        }

        stage('publish reports'){
            steps {
            
                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, includes: 'screenshot.png', keepAll: false, reportDir: '/var/lib/jenkins/workspace/medicure-pipeline', reportFiles: 'index.html', reportName: 'HTML Report', reportTitles: '', useWrapperFileDirectly: true])
            }
        }
        
        stage('Deployment on Prod-EKS Cluster'){
            when { expression {  params.action == 'create' } }
            steps{
                script{
                  
                  def apply = false

                  try{
                    input message: 'please confirm to deploy on prod-eks-cluster', ok: 'Ready to apply the config ?'
                    apply = true
                  }catch(err){
                    apply= false
                    currentBuild.result  = 'UNSTABLE'
                  }
                  if(apply){

                    sh """
                      kubectl apply -f prod-cluster-deployment.yaml
                    """
                  }
                }
            }
        } 

    }
}
