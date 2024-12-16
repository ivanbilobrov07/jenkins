pipeline {
    agent any
    environment {
        AZURE_SERVICE_PRINCIPAL_CREDS = credentials('azure-service-principal-credentials')
        AZURE_NEW_VM_CREDS = credentials('azure-new-vm-creds')
        AZURE_TENANT_ID = 'cead22b1-d045-44e9-a8bf-194b893191d8' 
        AZURE_NEW_RESOURCE_GROUP = 'for_jenkins'
        AZURE_LOCATION = 'eastus'
        AZURE_NEW_VM_NAME = 'jenkins-temp-vm'
    }
    stages {
        stage('Provision Azure VM') {
            steps {
                script {
                    sh '''
                    echo "1: Logging into Azure"
                    az login --service-principal -u $AZURE_SERVICE_PRINCIPAL_CREDS_USR -p $AZURE_SERVICE_PRINCIPAL_CREDS_PSW --tenant $AZURE_TENANT_ID
                  
                    echo "2: Create a resource group"
                    az group create --name $AZURE_NEW_RESOURCE_GROUP --location $AZURE_LOCATION
                    
                    echo "3: Create a VM"
                    az vm create \
                        --resource-group $AZURE_NEW_RESOURCE_GROUP \
                        --name $AZURE_NEW_VM_NAME \
                        --image Canonical:ubuntu-24_04-lts:server:latest \
                        --admin-username $AZURE_NEW_VM_CREDS_USR \
                        --admin-password $AZURE_NEW_VM_CREDS_PSW \
                    
                    echo "4: Open port 80"
                    az vm open-port \
                        --resource-group $AZURE_NEW_RESOURCE_GROUP \
                        --name $AZURE_NEW_VM_NAME \
                        --port 80
                    
                    echo "5: Getting VM IP"
                    PUBLIC_IP=$(az vm show -d -g $AZURE_NEW_RESOURCE_GROUP -n $AZURE_NEW_VM_NAME --query publicIps -o tsv)
                    echo "VM Public IP: $PUBLIC_IP"
                    echo $PUBLIC_IP > vm_ip.txt
                    
                    echo "6: Granting sudo privileges to $AZURE_NEW_VM_CREDS_USR"
                    az vm run-command invoke \
                        --resource-group $AZURE_NEW_RESOURCE_GROUP \
                        --name $AZURE_NEW_VM_NAME \
                        --command-id 'RunShellScript' \
                        --scripts "echo '$AZURE_NEW_VM_CREDS_USR ALL=(ALL) NOPASSWD: /bin/grep, /bin/echo' | sudo tee -a /etc/sudoers"
                    '''
                }
            }
        }
        stage('Install Apache') {
            steps {
                script {
                    def vmIp = readFile('vm_ip.txt').trim()
                    sshCommand remote: [
                        name: 'Azure-VM',
                        host: vmIp,
                        user: "$AZURE_NEW_VM_CREDS_USR",
                        password: "$AZURE_NEW_VM_CREDS_PSW",
                        allowAnyHosts: true
                    ], command: '''
                        sudo apt update
                        sudo apt install -y apache2
                        sudo systemctl start apache2
                        sudo systemctl enable apache2
                        echo "Hello World" | sudo tee /var/www/html/index.html > /dev/null
                    '''
                }
            }
        }
        stage('Test Apache with curl') {
            steps {
                script {
                    def vmIp = readFile('vm_ip.txt').trim()
                    sh """
                    echo "Testing Apache server"
                    curl http://$vmIp
                    curl http://$vmIp:80
                    curl http://$vmIp:80/test.html
                    """
                }
            }
        }
        stage('Check Apache Logs for Errors') {
            steps {
                script {
                    def vmIp = readFile('vm_ip.txt').trim()
                    sshCommand remote: [
                        name: 'Azure-VM',
                        host: vmIp,
                        user: "$AZURE_NEW_VM_CREDS_USR",
                        password: "$AZURE_NEW_VM_CREDS_PSW",
                        allowAnyHosts: true
                    ], command: '''
                        echo "Checking Apache error logs for 4xx and 5xx errors"
                        sudo grep "4[0-9][0-9]" /var/log/apache2/error.log || true
                        sudo grep "5[0-9][0-9]" /var/log/apache2/error.log || true
                        sudo grep "4[0-9][0-9]" /var/log/apache2/access.log || true
                        sudo grep "5[0-9][0-9]" /var/log/apache2/access.log || true
                    '''
                }
            }
        }
    }
    post {
        always {
            script {
                sh '''
                echo "5: Delete the VM and resource group"
                az vm delete --resource-group $AZURE_NEW_RESOURCE_GROUP --name $AZURE_NEW_VM_NAME --yes --no-wait
                az group delete --name $AZURE_NEW_RESOURCE_GROUP --yes --no-wait
                '''
            }
        }
    }
}
