pipeline {
  agent {
    label 'linux'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '60'))
    parallelsAlwaysFailFast()
  }
  environment {
    ROLE_ID="54c29b82-d415-dd33-288c-cb07ea43e16d"
    VAULT_ADDR="https://vault.copperdale.teknofile.net"

    PACKER_FILE = "ubuntu-20.04.json"
  }
  stages {
    stage("Setup Enviornment") {
      steps {
        script {
          env.EXIT_STATUS = ''
          env.COMMIT_SHA = sh(
            script: '''git rev-parse HEAD''',
            returnStdout: true).trim()
        }
      }
    }

    stage("Validate Template") {
      steps {
        echo "Testing to make sure that the json is right"
        sh "/usr/local/bin/packer validate ${PACKER_FILE}"
      }
    }

    // The Jenkins node is only allowed to create the wrapped secret ID
    // and with a wrap-ttl between 100s and 300s
    stage("Get AWS Creds") {
      steps {
        script {
          withCredentials([
            [
              $class: 'VaultTokenCredentialBinding',
              credentialsId: 'Jenkins_Node_Vault_AppRole',
              vaultAddr: 'https://vault.copperdale.teknofile.net'
            ]
          ]){

            env.WRAPPED_SID = sh(
              returnStdout: true,
              script: "vault write -field=wrapping_token -wrap-ttl=200s -f auth/pipeline/role/pipeline-approle/secret-id"
            )
            env.UNWRAPPED_SID = sh(
              returnStdout: true,
              script: "vault unwrap -field=secret_id ${WRAPPED_SID}"
            )
            env.VAULT_LOGIN_TOKEN = sh(
              returnStdout: true,
              script: "vault write -field=token auth/pipeline/login role_id=${ROLE_ID} secret_id=${UNWRAPPED_SID}"
            )
            env.VAULT_TOKEN1 = sh(
              returnStdout: true,
              script: "vault login -field=token ${VAULT_LOGIN_TOKEN}"
            )

            sh '''
              VAULT_TOKEN=${VAULT_TOKEN1} vault read aws/sts/tkfPipeline role_arn=arn:aws:iam::133530902744:role/tkfPipelineRole -format=json > /tmp/aws_creds.json
            '''
            env.AWS_ACCESS_KEY_ID = sh (
              returnStdout: true,
              script: "cat /tmp/aws_creds.json | jq -r .data.access_key"
            ).trim()
            env.AWS_SECRET_ACCESS_KEY = sh (
              returnStdout: true,
              script: "cat /tmp/aws_creds.json | jq -r .data.secret_key"
            ).trim()
            env.AWS_SESSION_TOKEN = sh (
              returnStdout: true,
              script: "cat /tmp/aws_creds.json | jq -r .data.security_token"
            ).trim()
          }
        }
      }
    }
        

    stage("Build the AMI") {
      steps {
        echo "Building the AMI"
          sh "/usr/local/bin/packer build ${PACKER_FILE}"
      }
    }
  }
}
