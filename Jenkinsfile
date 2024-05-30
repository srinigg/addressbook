pipeline {
    agent none

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "mymaven"
    }
    environment{
        BUILD_SERVER='ec2-user@172.31.8.129'
        IMAGE_NAME='devopstrainer/java-mvn-privaterepos:$BUILD_NUMBER'
        //DEPLOY_SERVER='ec2-user@172.31.14.15'
        ACM_IP='ec2-user@172.31.14.1'
       AWS_ACCESS_KEY_ID =credentials("AWS_ACCESS_KEY_ID")
        AWS_SECRET_ACCESS_KEY=credentials("AWS_SECRET_ACCESS_KEY")
        //created a new credential of type secret text to store docker pwd
        DOCKER_REG_PASSWORD=credentials("DOCKER_REG_PASSWORD")
    }
    stages {
        stage('Compile') {
           // agent {label "linux_slave"}
           agent any
            steps {              
              script{
                     echo "COMPILING"
                     sh "mvn compile"
              }             
            }
            
        }
        stage('Test') {
            agent any
            steps {           
              script{
                   echo "RUNNING THE TC"
                   sh "mvn test"
                }              
             
            }            
        
        post{
            always{
                junit 'target/surefire-reports/*.xml'
            }
        }
        }
        stage('Containerise-Build docker image') {
            agent any
            steps {             
                script{
                    sshagent(['slave2']) {
                    //echo "Creating the package"
                   //sh "mvn package"
                    withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    sh "scp -o StrictHostKeyChecking=no server-script.sh ${BUILD_SERVER}:/home/ec2-user"
                    sh "ssh -o StrictHostKeyChecking=no ${BUILD_SERVER} bash server-script.sh ${IMAGE_NAME}"
                    sh "ssh ${BUILD_SERVER} sudo docker login -u ${USERNAME} -p ${PASSWORD}"
                    sh "ssh ${BUILD_SERVER} sudo docker push ${IMAGE_NAME}"
                   
                }             
                }
                }
            }            
        }
        stage("Provision deploy server with TF"){
            // environment{
            //      ACCESS_KEY=credentials('ACCESS_KEY')
            //      SECRET_ACCESS_KEY=credentials('SECRET_ACCESS_KEY')
            // }
             agent any
                   steps{
                       script{
                           dir('terraform'){
                            sh 'aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}'
                            sh 'aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}'
                           sh "terraform init"
                           sh "terraform apply --auto-approve"
                           EC2_PUBLIC_IP = sh(
                            script: "terraform output instance-ip-0",
                            returnStdout: true
                           ).trim()
                       }
                       }
                   }
        }
         stage('Run ansible playbook on ACM'){
            agent any
            steps{
                script{
                    sshagent(['slave2']) {
                     sh "scp -o StrictHostKeyChecking=no ansible/* ${ACM_IP}:/home/ec2-user"
                   //copy the ansible target key on ACM as private key file
    withCredentials([sshUserPrivateKey(credentialsId: 'Ansible_target',keyFileVariable: 'keyfile',usernameVariable: 'user')]){ 
    sh "scp -o StrictHostKeyChecking=no $keyfile ${ACM_IP}:/home/ec2-user/.ssh/id_rsa"    
    }
    sh "ssh -o StrictHostKeyChecking=no ${ACM_IP} bash /home/ec2-user/prepare-ACM.sh ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} ${DOCKER_REG_PASSWORD} ${IMAGE_NAME}"
      }
        }
        }    
    }
    }
}
