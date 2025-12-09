pipeline {
  agent any

  environment {
    INVENTORY = "inventory.ini"
    PLAYBOOK  = "full-cluster-automation.yml"
    MASTER_IP = "10.0.2.15"
    SSH_CREDS = "sai"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Prepare SSH known_hosts') {
      steps {
        sshagent(credentials: [env.SSH_CREDS]) {
          sh '''
            mkdir -p ~/.ssh
            touch ~/.ssh/known_hosts

            # Extract IPs from inventory
            grep -Eo '^[0-9]+(\\.[0-9]+){3}' ${INVENTORY} > hosts.list || true

            while read host; do
              ssh-keyscan -H $host >> ~/.ssh/known_hosts 2>/dev/null || true
            done < hosts.list

            rm -f hosts.list
          '''
        }
      }
    }

    stage('Run Ansible Playbook') {
      steps {
        sshagent(credentials: [env.SSH_CREDS]) {
          sh '''
            ansible-playbook -i ${INVENTORY} ${PLAYBOOK}
          '''
        }
      }
    }

    stage('Fetch kubeconfig') {
      steps {
        sshagent(credentials: [env.SSH_CREDS]) {
          sh '''
            echo "Fetching kubeconfig from master..."
            scp -o StrictHostKeyChecking=no sai@${MASTER_IP}:/home/sai/.kube/config kubeconfig.yaml
          '''
        }
      }
    }

    stage('Archive kubeconfig') {
      steps {
        archiveArtifacts artifacts: 'kubeconfig.yaml', allowEmptyArchive: true
        echo "You can now download kubeconfig from Jenkins > Build Artifacts!"
      }
    }
  }

  post {
    success {
      echo "Cluster Created Successfully!"
    }
    failure {
      echo "Pipeline Failed!"
    }
  }
}
