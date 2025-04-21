pipeline {
    agent any

    environment {
        VAULT_TOKEN = credentials('vault-token')
        VM_HOST = 'oleg.aws.cts.care'
        VM_USER = 'ubuntu'
    }

    stages {
        stage('Install Git if Needed') {
            steps {
                script {
                    sh '''
                        if ! command -v git > /dev/null 2>&1; then
                            echo "📦 Git not found. Installing..."
                            apt-get update && apt-get install -y git
                        else
                            echo "✅ Git already installed."
                        fi
                    '''
                }
            }
        }

        stage('Install Ansible if Needed') {
            steps {
                script {
                    sh '''
                        if ! command -v ansible > /dev/null 2>&1; then
                            echo "📦 Ansible not found. Installing..."
                            apt-get update && apt-get install -y software-properties-common
                            apt-add-repository --yes --update ppa:ansible/ansible
                            apt-get install -y ansible
                        else
                            echo "✅ Ansible already installed."
                        fi
                    '''
                }
            }
        }

        stage('Get SSH Private Key from Vault') {
            steps {
                script {
                    sh 'mkdir -p ~/.ssh_temp && chmod 700 ~/.ssh_temp'

                    withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
                        sh '''
                            RESPONSE=$(curl --silent --header "X-Vault-Token: $VAULT_TOKEN" --request GET http://vault:8200/v1/secret/aws/pv-key)
                            echo "$RESPONSE" | sed 's/.*"value":"//' | sed 's/".*//' > ~/.ssh_temp/ssh_key.pem
                            sed -i 's/\\\\n/\\n/g' ~/.ssh_temp/ssh_key.pem
                            chmod 600 ~/.ssh_temp/ssh_key.pem
                        '''
                    }

                    echo "✅ SSH private key retrieved and saved."
                }
            }
        }

        stage('Run Ansible Playbook From Mounted Host Directory') {
            steps {
                script {
                    writeFile file: 'inventory.ini', text: """
                    [webserver]
                    ${VM_HOST} ansible_user=${VM_USER} ansible_ssh_private_key_file=~/.ssh_temp/ssh_key.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'
                    """

                    sh '''
                        echo "🚀 Running Ansible playbook..."
                        ansible-playbook -i inventory.ini /var/jenkins_home/nginx.yml
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'rm -rf ~/.ssh_temp inventory.ini'
                echo "🧹 Cleaned up temporary files."
            }
        }
    }

    post {
        always {
            sh 'rm -rf ~/.ssh_temp inventory.ini || true'
        }
    }
}
