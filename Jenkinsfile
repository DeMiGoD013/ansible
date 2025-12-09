pipeline {
  agent any

  parameters {
    choice(name: 'ACTION', choices: ['install','uninstall'], description: 'Choose operation')
  }

  environment {
    SSH_CREDENTIALS_ID = 'sai'        // set this to your Jenkins SSH credential id
    ANSIBLE_LOG = 'ansible-run.log'
    MASTER_IP = '10.0.2.15'           // change if different
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install dependencies (optional)') {
      steps {
        sh 'ansible --version || (pip install ansible && ansible --version)'
      }
    }

    stage('Run playbook') {
      steps {
        sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
          script {
            def playbook = (params.ACTION == 'install') ? 'install-rke2.yml' : 'uninstall-rke2.yml'
            sh """
              echo "Running playbook: ${playbook}"
              ansible-playbook -i inventory.ini ${playbook} | tee ${ANSIBLE_LOG}
            """
          }
        }
      }
    }

    stage('Fetch kubeconfig from Master') {
      when { expression { params.ACTION == 'install' } }
      steps {
        sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
          sh '''
            echo "Fetching kubeconfig from master..."
            scp -o StrictHostKeyChecking=no sai@${MASTER_IP}:/etc/rancher/rke2/rke2.yaml master-kubeconfig.yaml || \
            scp -o StrictHostKeyChecking=no sai@${MASTER_IP}:/home/sai/rke2.yaml master-kubeconfig.yaml
            echo "Patching kubeconfig server IP..."
            sed -i "s/127.0.0.1/${MASTER_IP}/g" master-kubeconfig.yaml
            echo "Done."
          '''
        }
      }
    }

    stage('Archive logs & kubeconfig') {
      steps {
        archiveArtifacts artifacts: "${ANSIBLE_LOG}, master-kubeconfig.yaml", allowEmptyArchive: true
      }
    }
  }

  post {
    always {
      sh "tail -n 200 ${ANSIBLE_LOG} || true"
    }
    success {
      echo "Pipeline finished successfully."
    }
    failure {
      echo "Pipeline failed — check the archived ${ANSIBLE_LOG} for details."
    }
  }
}
