pipeline {
  agent { 
    // uses a Docker image with Ansible installed
    docker {
      image 'williamyeh/ansible:alpine3'   // lightweight image with ansible + ssh client
      args '-u root:root'                  // optional: run as root if needed in your Jenkins setup
    }
  }

  parameters {
    choice(name: 'ACTION', choices: ['install','uninstall'], description: 'Choose whether to install or uninstall RKE2')
    string(name: 'INVENTORY_PATH', defaultValue: 'inventory.ini', description: 'Path to Ansible inventory (relative to repo root)')
    string(name: 'EXTRA_VARS', defaultValue: '', description: 'Any extra vars to pass to ansible-playbook (e.g. "rke2_version=1.27.0")')
    string(name: 'ANSIBLE_LIMIT', defaultValue: '', description: 'Limit to subset of hosts (optional), e.g. "MASTER" or "worker1"')
    booleanParam(name: 'DRY_RUN', defaultValue: false, description: 'If true, perform ansible-playbook --check')
  }

  environment {
    // credentialsId must match an SSH private key credential created in Jenkins (see instructions below)
    SSH_CREDENTIALS_ID = 'rke2_ssh' 
    ANSIBLE_LOG = 'ansible-run.log'
    // Ensure Ansible doesn't prompt
    ANSIBLE_HOST_KEY_CHECKING = 'False'
  }

  stages {
    stage('Prepare') {
      steps {
        // show repo contents for debugging
        sh 'echo "Workspace contents:" && ls -la'
        // print chosen parameters
        sh 'echo "ACTION=$ACTION INVT=$INVENTORY_PATH LIMIT=$ANSIBLE_LIMIT DRY_RUN=$DRY_RUN"'
      }
    }

    stage('Checkout') {
      steps {
        // default checkout; Jenkins will checkout repo for Multibranch Pipeline or Pipeline from SCM
        checkout scm
      }
    }

    stage('Install dependencies (if needed)') {
      steps {
        // If you're using a container that already has ansible this is normally not needed.
        // But we'll check ansible version for clarity.
        sh 'ansible --version || (pip install ansible && ansible --version)'
      }
    }

    stage('Run playbook') {
      steps {
        // Use ssh-agent wrapper to load private key credential so ansible can SSH to targets.
        // Requires "SSH Agent" plugin in Jenkins and a credential of type "SSH Username with private key".
        sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
          script {
            // select playbook
            def playbook = (params.ACTION == 'install') ? 'install-rke2.yml' : 'uninstall-rke2.yml'

            // build command
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
  }

  post {
    success {
      echo "Playbook finished successfully."
    }
    failure {
      echo "Playbook failed — check the archived ${ANSIBLE_LOG} for details."
    }
    always {
      // print last 200 lines of log for quick view
      sh "tail -n 200 ${ANSIBLE_LOG} || true"
    }
  }
}
