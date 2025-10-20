pipeline {
  agent any
  environment {
    BASTION_INSTANCE_ID = credentials('bastion-id')
    ANSIBLE_IP = credentials('ansible-ip')  // Keeping IP since we need it for port 22 connection
    AWS_REGION = 'eu-west-1'
  }
  stages {
    stage ('Deploying to Stage Environment') {
      steps {
          script {
            // Start SSM session to bastion with port forwarding for SSH (port 22)
            sh '''
              aws ssm start-session \
                --target ${BASTION_INSTANCE_ID} \
                --region ${AWS_REGION} \
                --document-name AWS-StartPortForwardingSession \
                --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                &
              sleep 5  # Wait for port forwarding to establish
            '''
            
            // SSH through the tunnel to Ansible server on port 22
            sshagent(['ansible-key']) {
              sh '''
                ssh -o StrictHostKeyChecking=no \
                    -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                    ubuntu@${ANSIBLE_IP} \
                    "ansible-playbook /etc/ansible/playbooks/deploy-stage.yml"
              '''
            }
            
            // Terminate the SSM session
            sh 'pkill -f "aws ssm start-session"'
          }
      }
    }
    
    stage ('Slack Notification for prod') {
      steps {
        slackSend channel: 'Cloudhight', message: 'New Stage Deployment', teamDomain: '8th-sept-2025-sock-shop-e-commerce-project-eu-team1', tokenCredentialId: 'slack'
      }
    }
    
    stage ('DAST Scan') {
      steps {
        sh '''
          chmod 777 $(pwd)
          docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://stage.work-experience-2025.buzz -g gen.conf -r testreport.html || true
        '''
      }
    }
    
    stage ('Prompt for Approval') {
      steps {
        timeout(activity: true, time: 5) {
          input message: 'Review before approval', submitter: 'admin'
        }
      }
    }
    
    stage ('Deploying to Prod Environment') {
      steps {
          script {
            sh '''
              aws ssm start-session \
                --target ${BASTION_INSTANCE_ID} \
                --region ${AWS_REGION} \
                --document-name AWS-StartPortForwardingSession \
                --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                &
              sleep 5
            '''
            
            sshagent(['ansible-key']) {
              sh '''
                ssh -o StrictHostKeyChecking=no \
                    -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                    ubuntu@${ANSIBLE_IP} \
                    "ansible-playbook /etc/ansible/playbooks/deploy-prod.yml"
              '''
            }
            
            sh 'pkill -f "aws ssm start-session"'
          }
      }
    }
    
    stage ('Slack Notification') {
      steps {
        slackSend channel: 'Cloudhight', message: 'New Production Deployment', teamDomain: '8th-sept-2025-sock-shop-e-commerce-project-eu-team1', tokenCredentialId: 'slack'
      }
    }
  }
}
