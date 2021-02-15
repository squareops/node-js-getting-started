def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {
  agent any
  tools {nodejs "nodejs"}
  environment {
    // Put your environment variables
    doError = '0'
    DOCKER_REPO = "421320058418.dkr.ecr.eu-central-1.amazonaws.com/jenkins-demo"
    ECR_REPO = "jenkins-demo"
    AWS_REGION = "eu-central-1"
    HELM_RELEASE_NAME = "node-demo"
    CLUSTER_NAME = "test-squareops-eks"
    COUNT_VALUE = '1'
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }
  // // Check every minute for changes
  // triggers {
  //   pollSCM('*/1 * * * *')
  // }
  stages {
    // Check code quality using sonarqube
    stage('Code Quality Check via SonarQube') {
      steps {
        script {
          def scannerHome = tool 'sonarqube';
            withSonarQubeEnv("sonarqube-jenkins") {
            sh "${tool("sonarqube")}/bin/sonar-scanner \
            -Dsonar.projectKey=jenkins-scan"
          }
        }
      }
    }
    stage('Abort pipeline if SonarQube Fails') {
      steps {
        waitForQualityGate abortPipeline: true
      }
    }  
    // Build container image
    stage('Build') {
      agent {
        kubernetes {
          label 'jenkinsrun'
          defaultContainer 'dind'
          yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:18.05-dind
    securityContext:
      privileged: true
    volumeMounts:
      - name: dind-storage
        mountPath: /var/lib/docker
  volumes:
    - name: dind-storage
      emptyDir: {}
"""
        }
      }
      steps {
        withAWS(credentials: 'jenkins-user') {
        container('dind') {
          script {
            sh '''
            echo 'Creating Artifact'
            apk --update add ca-certificates wget python curl tar jq
            apk -Uuv add make groff less python py-pip
            pip install awscli
            $(aws ecr get-login --region ${AWS_REGION} --no-include-email)
            docker build --network=host -t ${DOCKER_REPO}:v${BUILD_NUMBER} .
            docker push ${DOCKER_REPO}:v${BUILD_NUMBER}
            VERSION=v3.2.4
            echo $VERSION
            FILENAME=helm-${VERSION}-linux-amd64.tar.gz
            HELM_URL=https://get.helm.sh/${FILENAME}
            echo $HELM_URL
            curl -o /tmp/$FILENAME ${HELM_URL} \
            && tar -zxvf /tmp/${FILENAME} -C /tmp \
            && mv /tmp/linux-amd64/helm /bin/helm
            sleep 30
            count=$(aws ecr describe-image-scan-findings --repository-name ${ECR_REPO} --image-id imageTag=v${BUILD_NUMBER} --region ${AWS_REGION} | jq -r '.imageScanFindings.findings[]?.severity' | grep "CRITICAL" | wc -l)
            value=${COUNT_VALUE}
            if [ $count -lt $value ]
            then
              exit 1
            fi
            echo 'Start Deploying'
            aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
            helm upgrade --install ${HELM_RELEASE_NAME} ./helm \
            --set image.repository=${DOCKER_REPO} --set image.tag=v${BUILD_NUMBER}
            '''
          } //script
        } //container
        } //withAWS
      } //steps
    }
    stage("Deploy to Stage?") {
      agent {
        kubernetes {
          label 'jenkinsrun'
          defaultContainer 'dind'
          yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:18.05-dind
    securityContext:
      privileged: true
    volumeMounts:
      - name: dind-storage
        mountPath: /var/lib/docker
  volumes:
    - name: dind-storage
      emptyDir: {}
"""
        }
      }
      steps {
        script {
          def userInput = input(
              id: 'userInput', message: 'Let\'s Promote?', parameters: [
                  [$class: 'TextParameterDefinition', defaultValue: 'stage', description: 'Environment', name: 'Env']
              ]
          )
          echo ("Env: "+userInput)

          if ("$userInput" == "stage") {
            stage ("Deploying to Stage") {
              withAWS(credentials: 'jenkins-user') {
                container('dind') {
                  script {
                    sh '''
                    apk --update add ca-certificates wget python curl tar jq
                    apk -Uuv add make groff less python py-pip
                    pip install awscli
                    aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
                    VERSION=v3.2.4
                    echo $VERSION
                    FILENAME=helm-${VERSION}-linux-amd64.tar.gz
                    HELM_URL=https://get.helm.sh/${FILENAME}
                    echo $HELM_URL
                    curl -o /tmp/$FILENAME ${HELM_URL} \
                    && tar -zxvf /tmp/${FILENAME} -C /tmp \
                    && mv /tmp/linux-amd64/helm /bin/helm
                    helm upgrade --install ${HELM_RELEASE_NAME} ./helm \
                    --set image.repository=${DOCKER_REPO} --set image.tag=v${BUILD_NUMBER} -n stage
                    '''
                  }
                }    
              }
            }
          }
        }
      }
    } 
// Slack notification configuration
  // stage('Error') {
  //   // when doError is equal to 1, return an error
  //   when {
  //       expression { doError == '1' }
  //   }
  //   steps {
  //       echo "Failure :("
  //       error "Test failed on purpose, doError == str(1)"
  //   }
  // }
  // stage('Success') {
  //   // when doError is equal to 0, just print a simple message
  //   when {
  //       expression { doError == '0' }
  //   }
  //   steps {
  //       echo "Success :)"
  //   }
  // }
  } //stages
    // Post-build actions
  // post {
  //     always {
  //         slackSend channel: '#test123',
  //             color: COLOR_MAP[currentBuild.currentResult],
  //             message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} More info at: https://jenkins.squareops.com/blue/organizations/jenkins/$JOB_BASE_NAME/detail/$JOB_BASE_NAME/$BUILD_NUMBER"
  //     }
  // }
} //pipeline