pipeline {
    agent any

    environment {
        PIPELINE_NAME = "phonebook"
        TF_FOLDER     = "infra-tf"
        K8S_FOLDER    = "k8s"
    }

    stages {
        stage('Create Infrastructure for the App') {
            steps {
                sh 'az login --identity'
                dir("/var/lib/jenkins/workspace/${PIPELINE_NAME}/${TF_FOLDER}") {
                    echo 'Creating Infrastructure for the App'
                    sh 'terraform init'
                    sh 'terraform apply --auto-approve'
                }
            }
        }

        stage('Create rule on AKS NSG') {
            steps {
                dir("/var/lib/jenkins/workspace/${PIPELINE_NAME}/${TF_FOLDER}") {
                    echo 'Injecting Terraform outputs into NSG rule creation command'
                    script {
                        env.AKS_NODE_RG_NAME = sh(script: 'terraform output -raw aks_node_rg_name', returnStdout:true).trim()
                        env.AKS_NSG_NAME = sh(script: "az network nsg list --resource-group ${AKS_NODE_RG_NAME} --query \"[?contains(name, 'aks')].[name]\" --output tsv", returnStdout:true).trim()
                    }
                    sh "az network nsg rule create --nsg-name ${AKS_NSG_NAME} --resource-group ${AKS_NODE_RG_NAME} --name open3000130002 --access Allow --priority 100 --destination-port-ranges 30001-30002"
                }
            }
        }

        stage('Connect to AKS') {
            steps {
                dir("/var/lib/jenkins/workspace/${PIPELINE_NAME}/${TF_FOLDER}") {
                    echo 'Injecting Terraform output into connection command'
                    script {
                        env.AKS_NAME = sh(script: 'terraform output -raw aks_name', returnStdout:true).trim()
                        env.RG_NAME = sh(script: 'terraform output -raw rg_name', returnStdout:true).trim()
                    }
                    sh "az aks get-credentials --resource-group ${RG_NAME} --name ${AKS_NAME}"
                }
            }
        }

        stage('Deploy K8s files') {
            steps {
                dir("/var/lib/jenkins/workspace/${PIPELINE_NAME}/${K8S_FOLDER}") {
                    sh 'kubectl apply -f .'
                }
            }
        }

        stage('Destroy the Infrastructure') {
            steps{
                timeout(time:5, unit:'DAYS'){
                    input message:'Do you want to destroy the infrastructure?'
                }
                dir("/var/lib/jenkins/workspace/${PIPELINE_NAME}/${TF_FOLDER}") {
                    sh 'terraform destroy --auto-approve'
                }
            }
        }
    }
}
