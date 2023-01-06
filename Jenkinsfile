pipeline {
    agent any
    parameters {
        choice(name: 'ACTION', choices: ['Select', 'Deploy', 'Undeploy'], description: 'Choose the action to Install or Uninstall FDB.')
        string(name: 'aws_region', defaultValue: 'ap-south-1', description: 'AWS Region in which cluster needs to be deployed.')
        string(name: 'aws_act_id', defaultValue: '', description: 'AWS account in which service needs to be deployed.')
        password(name: 'atlas_public_key', defaultValue: '', description: 'MongoDB Atlas public key for provisioning.')
        password(name: 'atlas_private_key', defaultValue: '', description: 'MongoDB Atlas private key for provisioning.')
        string(name: 'atlas_region', defaultValue: 'AP_SOUTH_1', description: 'Atlas Region in which MongoDB cluster needs to be deployed.')
        string(name: 'db_user', defaultValue: '', description: 'User name for creating in MongoDB Atlas.')
        password(name: 'db_password', defaultValue: '', description: 'User Password.')
        string(name: 'org_id', defaultValue: '', description: 'Organiazation id in which org to create to creating in MongoDB Atlas.')
        string(name: 'atlas_cluster_name', defaultValue: '', description: 'Atlas cluster name')
        string(name: 'aws_vpc_cidr', defaultValue: '', description: 'AWS VPC cidr block')
        string(name: 'subnet_a_cidr', defaultValue: '', description: 'AWS VPC subnet a cidr block')
        string(name: 'subnet_b_cidr', defaultValue: '', description: 'AWS VPC subnet b cidr block')
    }
    environment {
        BRANCH_NAME_ENV = "main"
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_SESSION_TOKEN = credentials('AWS_SESSION_TOKEN')
        TF_LOG="TRACE"
    }
    stages {
        stage('checkout') {
            steps {
                git branch: "${BRANCH_NAME_ENV}" ,credentialsId: 'git-cred-for-MongoDB',url: "https://github.ibm.com/Irfan-Mohiuddin/aws-atlas-terraform.git"
            }
        }
        stage('TLS management') {
            when {
                expression { params.ACTION == 'Select' }
            }
            steps {
                script{
                    currentBuild.result = 'ABORTED'
                    error('Please select what ACTION needs to be performed')
                }
            }
        }
        stage('Validate cluster name') {
            when {
                expression { params.ACTION == 'Deploy' }
            }
            steps{
                script{
                    def res = 0
                    res = sh(script: '''
                        clusters_json="$(curl -X GET --silent --digest --user "${atlas_public_key}:${atlas_private_key}" "https://cloud.mongodb.com/api/atlas/v1.0/clusters/" | jq ".results[].clusters[].name" )"
                        echo $clusters_json
                        if [ ! -z $clusters_json ]; then
                            if echo "$clusters_json" | grep -E ${atlas_cluster_name} 1>/dev/null 2>&1; then
                                echo "exists"
                                exit 1
                            else
                                echo "not exists"
                                exit 0
                            fi
                        else
                            echo "no cluster exist"
                            exit 0
                        fi
                    '''
                    , returnStatus:true)
                    if (res != 0) {
                        currentBuild.result = 'ABORTED'
                        error('Provided cluster name already exist')
                    }
                }
            }
        }
        stage('Terraform init') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE'){
                sh '''
                    pwd
                    ls -la
                    rm -rf .terraform .terraform.lock.hcl terraform.tfstate.d
                    ls -la
                    terraform init -backend-config="key=${db_user}/${atlas_cluster_name}/terraform.tfstate"
                   '''
                }
            }    
        }
        stage('Terraform Plan') {
            when {
                expression { params.ACTION == 'Deploy' }     
            }
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE'){
                sh '''
                    echo "${db_user}/${atlas_cluster_name}/terraform.tfstate"
                    echo "terraform plan"
                    terraform plan \
                      -var access_key=${AWS_ACCESS_KEY_ID} \
                      -var secret_key=${AWS_SECRET_ACCESS_KEY} \
                      -var aws_region=${aws_region} \
                      -var token=${AWS_SESSION_TOKEN} \
                      -var public_key=${atlas_public_key} \
                      -var private_key=${atlas_private_key} \
                      -var atlas_region=${atlas_region} \
                      -var atlas_dbuser=${db_user} \
                      -var atlas_dbpassword=${db_password} \
                      -var atlasorgid=${org_id} \
                      -var aws_account_id=${aws_act_id} \
                      -var cluster_name=${atlas_cluster_name} \
                      -var aws_cidr=${aws_vpc_cidr} \
                      -var subnet_a_cidr=${subnet_a_cidr} \
                      -var subnet_b_cidr=${subnet_b_cidr} \
                      -out my.plan
                   '''
                }
            }
        }
        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'Deploy' }     
            }            
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE'){
                sh '''
                     echo "terraform Apply"
                     terraform apply "my.plan"
                   '''
                }     
            }
        }
        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'Undeploy' }     
            }
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE'){
                sh '''
                     echo "terraform Destroy"
                     terraform destroy \
                      -var access_key=${AWS_ACCESS_KEY_ID} \
                      -var secret_key=${AWS_SECRET_ACCESS_KEY} \
                      -var aws_region=${aws_region} \
                      -var token=${AWS_SESSION_TOKEN} \
                      -var public_key=${atlas_public_key} \
                      -var private_key=${atlas_private_key} \
                      -var atlas_region=${atlas_region} \
                      -var atlas_dbuser=${db_user} \
                      -var atlas_dbpassword=${db_password} \
                      -var atlasorgid=${org_id} \
                      -var aws_account_id=${aws_act_id} \
                      -var cluster_name=${atlas_cluster_name} \
                      -var aws_cidr=${aws_vpc_cidr} \
                      -var subnet_a_cidr=${subnet_a_cidr} \
                      -var subnet_b_cidr=${subnet_b_cidr} \
                      --auto-approve
                   '''
                }
            }
        }
    }
}
