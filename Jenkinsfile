pipeline {

  agent any

  parameters {
    choice(name: 'ACTION', choices: ['install','uninstall'], description: 'Choose whether to install or uninstall RKE2')
    string(name: 'INVENTORY_PATH', defaultValue: 'inventory.ini', description: 'Path to Ansible inventory (relative to repo root)')
    string(name: 'EXTRA_VARS', defaultValue: '', description: 'Any extra vars to pass to ansible-playbook (e.g. "rke2_version=1.27.0")')
    string(name: 'ANSIBLE_LIMIT', defaultValue: '', description: 'Limit to subset of hosts (optional), e.g. "MASTER" or "worker1"')
    booleanParam(name: 'DRY_RUN', defaultValue: false, description: 'If true, perform ansible-playbook --check')
  }

  environment {
    SSH_CREDENTIALS_ID = 'rke2_ssh'
    ANSIBLE_LOG = 'ansible-run.log'
    ANSIBLE_HOST_KEY_CHECKING = 'False'
  }

  stages {

    stage('Prepare') {
      steps {
        sh 'echo "Workspace contents:" && ls -la'
        sh 'echo "ACTION=$ACTION INVT=$INVENTORY_PATH LIMIT=$ANSIBLE_LIMIT DRY_RUN=$DRY_RUN"'
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
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

    stage('Archive logs') {
      steps {
        archiveArtifacts artifacts: "${ANSIBLE_LOG}", allowEmptyArchive: true
      }
    }

  } // end stages

  post {
    success {
      echo "Playbook finished successfully."
    }
    failure {
      echo "Playbook failed — check the archived ${ANSIBLE_LOG} for details."
    }
    always {
      sh "tail -n 200 ${ANSIBLE_LOG} || true"
    }
  }

} // end pipeline
