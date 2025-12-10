pipeline {

  agent any

  parameters {
    choice(name: 'ACTION', choices: ['install','uninstall'], description: 'Choose whether to install or uninstall RKE2')
    string(name: 'INVENTORY_PATH', defaultValue: 'inventory.ini', description: 'Path to Ansible inventory (relative to repo root)')
    string(name: 'EXTRA_VARS', defaultValue: '', description: 'Any extra vars to pass to ansible-playbook')
    string(name: 'ANSIBLE_LIMIT', defaultValue: '', description: 'Limit to subset of hosts (optional)')
    booleanParam(name: 'DRY_RUN', defaultValue: false, description: 'If true, perform ansible-playbook --check')
  }

  environment {
    SSH_CREDENTIALS_ID = 'rke2_ssh'
    ANSIBLE_LOG = 'ansible-run.log'
    MASTER_IP = '10.0.2.15'
    ANSIBLE_HOST_KEY_CHECKING = 'False'
  }

  stages {

    stage('Prepare') {
      steps {
        sh 'echo "Workspace contents:" && ls -la'
        sh 'echo ACTION=$ACTION INVT=$INVENTORY_PATH LIMIT=$ANSIBLE_LIMIT DRY_RUN=$DRY_RUN'
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare SSH known_hosts') {
      steps {
        sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
          sh '''
            mkdir -p ~/.ssh
            touch ~/.ssh/known_hosts

            grep -Eo '^[0-9]+(\\.[0-9]+){3}' ${INVENTORY_PATH} > hosts.list || true
            grep -Eo 'ansible_host=[0-9]+(\\.[0-9]+){3}' ${INVENTORY_PATH} | sed 's/ansible_host=//' >> hosts.list || true

            sort -u hosts.list | while read host; do
              ssh-keyscan -H "$host" >> ~/.ssh/known_hosts 2>/dev/null || true
            done

            rm -f hosts.list
          '''
        }
      }
    }

    stage('Install dependencies (if needed)') {
      steps {
        sh 'ansible --version || (pip install ansible && ansible --version)'
      }
    }

    stage('Run playbook') {
      steps {
        sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
          script {
            def playbook = (params.ACTION == 'install') ? 'install-rke2.yml' : 'uninstall-rke2.yml'
            def extra = params.EXTRA_VARS?.trim() ? "--extra-vars '${params.EXTRA_VARS}'" : ''
            def limit = params.ANSIBLE_LIMIT?.trim() ? "-l '${params.ANSIBLE_LIMIT}'" : ''
            def check = params.DRY_RUN ? '--check' : ''

            sh """
              echo "Running playbook: ${playbook}"
              ansible-playbook -i ${params.INVENTORY_PATH} ${playbook} ${extra} ${limit} ${check} | tee ${ANSIBLE_LOG}
            """
          }
        }
      }
    }

    stage('Fetch kubeconfig') {
      when {
        expression { params.ACTION == 'install' }
      }
      steps {
        sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
          sh '''
            echo "Fetching kubeconfig from master..."

            # Copy kubeconfig to Jenkins workspace
            scp -o StrictHostKeyChecking=no sai@${MASTER_IP}:/home/sai/.kube/config kubeconfig.yaml

            # Deploy kubeconfig on Jenkins agent
            mkdir -p ~/.kube
            cp kubeconfig.yaml ~/.kube/config
            chmod 600 ~/.kube/config

            echo "Kubeconfig installed — verifying connection..."
            kubectl get nodes || true
          '''
        }
      }
    }

    stage('Archive logs') {
      steps {
        archiveArtifacts artifacts: "${ANSIBLE_LOG}", allowEmptyArchive: true
      }
    }

    stage('Archive kubeconfig') {
      when {
        expression { params.ACTION == 'install' }
      }
      steps {
        archiveArtifacts artifacts: "kubeconfig.yaml", allowEmptyArchive: false
      }
    }

  } // end stages

  post {
    success {
      echo "Playbook finished successfully."
    }
    failure {
      echo "Playbook failed — check ansible-run.log for details."
    }
    always {
      sh "tail -n 200 ${ANSIBLE_LOG} || true"
    }
  }

} // end pipeline
