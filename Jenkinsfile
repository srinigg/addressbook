pipeline {
    agent none
    // tools{
    //     jdk 'myjava'
    //     maven 'mymaven'
    // }
     environment{
        IMAGE_NAME='devopstrainer/java-mvn-privaterepos'
        DEV_SERVER_IP='ec2-user@172.31.39.165'
        ACM_IP='ec2-user@172.31.34.250'
        APP_NAME='java-mvn-app'
        AWS_ACCESS_KEY_ID =credentials("ACCESS_KEY")
        AWS_SECRET_ACCESS_KEY=credentials("SECRET_ACCESS_KEY")
        DOCKER_REG_PASSWORD=credentials("DOCKER_REG_PASSWORD")
    }
    stages {
        // stage('COMPILE') {
        //     agent any
        //     steps {
        //         script{
        //             echo "COMPILING THE CODE"
        //             sh 'mvn compile'
        //         }
        //                   }
        //     }
        // stage('UNITTEST'){
        //     agent any
        //     steps {
        //         script{
        //             echo "RUNNING THE UNIT TEST CASES"
        //             sh 'mvn test'
        //         }
              
        //     }
        //     post{
        //         always{
        //             junit 'target/surefire-reports/*.xml'
        //         }
        //     }
        //     }
        stage('PACKAGE+BUILD DOCKER IMAGE ON BUILD SERVER'){
            agent any
           steps{
            script{
            sshagent(['slave2']) {
        withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                     echo "PACKAGING THE CODE"
                     sh "scp -o StrictHostKeyChecking=no server-script.sh ${DEV_SERVER_IP}:/home/ec2-user"
                     sh "ssh -o StrictHostKeyChecking=no ${DEV_SERVER_IP} 'bash ~/server-script.sh ${IMAGE_NAME} ${BUILD_NUMBER}'"
                //     sh "ssh ${DEV_SERVER_IP} sudo docker build -t  ${IMAGE_NAME} /home/ec2-user/addressbook"
                    sh "ssh ${DEV_SERVER_IP} sudo docker login -u $USERNAME -p $PASSWORD"
                    sh "ssh ${DEV_SERVER_IP} sudo docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                    }
                    }
                }
            }
        }
        stage("Provision anisble target server with TF"){
            agent any
                   steps{
                       script{
                           dir('terraform'){
                           sh "terraform init"
                           sh "terraform apply --auto-approve"
                           ANSIBLE_TARGET_PUBLIC_IP = sh(
                            script: "terraform output ec2-ip",
                            returnStdout: true
                           ).trim()
                         echo "${ANSIBLE_TARGET_PUBLIC_IP}"   
                       }
                       }
                   }
        }
        stage("RUN ansible playbook on ACM"){
            agent any
            steps{
            script{
                echo "copy ansible files on ACM and run the playbook"
               sshagent(['slave2']) {
    sh "scp -o StrictHostKeyChecking=no ansible/* ${ACM_IP}:/home/ec2-user"
    withCredentials([sshUserPrivateKey(credentialsId: 'ANSIBLE_TARGET_KEY',keyFileVariable: 'keyfile',usernameVariable: 'user')]){ 
    sh "scp -o StrictHostKeyChecking=no $keyfile ${ACM_IP}:/home/ec2-user/.ssh/id_rsa"    
    }
    sh "ssh -o StrictHostKeyChecking=no ${ACM_IP} bash /home/ec2-user/prepare-ACM.sh ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} ${DOCKER_REG_PASSWORD} ${IMAGE_NAME}"
        }
        }
        }    
    }
    }
}
