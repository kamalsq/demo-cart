def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]



pipeline {
  environment {
    doError = '0'
    AWS_ACCOUNT_ID = 271251951598
    AWS_REGION = "us-east-2"
    DOCKER_REPO_BASE_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    DOCKER_REPO_NAME = """${sh(
                returnStdout: true,
                script: 'basename=$(basename $GIT_URL) && echo ${basename%.*}'
            ).trim()}"""
    HELM_CHART_GIT_REPO_URL = "https://gitlab.com/sq-ia/ref/msa-app/helm.git"
    HELM_CHART_GIT_BRANCH = "qa"
    GIT_USER_EMAIL = "pipelines@squareops.com"
    GIT_USER_NAME = "squareops"  
    DEPLOYMENT_STAGE = """${sh(
                returnStdout: true,
                script: 'echo ${GIT_BRANCH#origin/}'
            ).trim()}"""
    last_started_build_stage = ""   
    IMAGE_NAME="${DOCKER_REPO_BASE_URL}/${DOCKER_REPO_NAME}/stg"
    def BUILD_DATE = sh(script: "echo `date +%d_%m_%Y`", returnStdout: true).trim();
    def scannerHome = tool 'SonarqubeScanner';
  }

  options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
  }

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
              image: squareops/jenkins-build-agent:latest
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
  
  stages {
    stage ('Static code Analysis') {
      steps {
// NodeJS Scan
        script { 
            sh '''
            echo "NodejsScan"
            nodejsscan --directory `pwd`
            '''
        }
// Sonarqube Analysis
        withSonarQubeEnv ('SonarqubeScanner') {
            sh 'echo SonarQube Analysis'
            sh 'echo ${scannerHome}'
		sh '${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${DOCKER_REPO_NAME}-${DEPLOYMENT_STAGE} -Dsonar.qualitygate=my_custom_quality_gate''
         }
// Sonarqube Quality Gate
         sh 'echo SonarQube Quality gate'
         timeout(time: 1, unit: 'HOURS') {
          waitForQualityGate abortPipeline: true
       }
// git-secret scan
//         script { 
//             sh '''
//             echo 'git secret scanning'
//             git config --global --add safe.directory /home/jenkins/agent/workspace/${DOCKER_REPO_NAME}
//             git secrets --register-aws
//             git-secrets --scan
//             '''
//         }
      }
     }

    stage('Build Docker Image') {   
      agent {
        kubernetes {
          label 'kaniko'
          yaml """
          apiVersion: v1
          kind: Pod
          metadata:
            name: kaniko              
          spec:
            restartPolicy: Never
            containers:
            - name: kaniko
              image: gcr.io/kaniko-project/executor:debug
              command:
              - /busybox/cat
              tty: true 
          """
        }
      }  
      steps {
       container('kaniko'){
            script {
              last_started = env.STAGE_NAME
              echo 'Build start'              
              sh '''/kaniko/executor --dockerfile Dockerfile  --context=`pwd` --destination=${IMAGE_NAME}:${BUILD_NUMBER}-${BUILD_DATE} --no-push --oci-layout-path `pwd`/build/ --tarPath `pwd`/build/${DOCKER_REPO_NAME}-${BUILD_NUMBER}.tar
              '''               
            }   
            stash includes: 'build/*.tar', name: 'image'          
        }
      }
    }
    stage('Scan Docker Image') {
      when {		
	    anyOf {
                        branch 'main';
                        branch 'development'
	    }            
	   }
      agent {
        kubernetes {           
            containerTemplate {
              name 'trivy'
              image 'aquasec/trivy:0.35.0'
              command 'sleep'
              args 'infinity'
            }
        }
      }
      options { skipDefaultCheckout() }
      steps {
        container('trivy') {
           script {
              last_started = env.STAGE_NAME
              echo 'Scan with trivy'    
              unstash 'image'          
              sh '''
              apk add jq
              trivy image --ignore-unfixed -f json -o scan-report.json --input build/${DOCKER_REPO_NAME}-${BUILD_NUMBER}.tar
              '''
              echo 'archive scan report'
              archiveArtifacts artifacts: 'scan-report.json'
              echo 'Docker Image Vulnerability Scanning'
              high = sh (
                   script: 'cat scan-report.json | jq .Results[].Vulnerabilities[].Severity | grep HIGH | wc -l',
                   returnStdout: true
              ).trim()
              echo "High: ${high}"
             
             critical = sh (
                  script: 'cat scan-report.json | jq .Results[].Vulnerabilities[].Severity | grep CRITICAL | wc -l',
                   returnStdout: true
              ).trim()
              echo "Critical: ${critical}"
           }
         }
      } 
    }    

    stage('Push to ECR') {
      when {
                branch 'main'
            }
       agent {
          kubernetes { 
          yaml """
          apiVersion: v1
          kind: Pod
          metadata:
            name: kaniko              
          spec:
            restartPolicy: Never
            containers:
            - name: kaniko
              image: gcr.io/kaniko-project/executor:debug
              command:
              - /busybox/cat
              tty: true 
          """  
            }
        }
      //  options { skipDefaultCheckout() }
       steps {        
         container('kaniko') {
           script {
              echo 'push to ecr step start'
              if ( "$high" < 500 && "$critical" < 80 ) {
                withAWS(credentials: 'jenkins-demo-aws') {  
                sh '''                                   
                /kaniko/executor --dockerfile Dockerfile  --context=`pwd` --destination=${IMAGE_NAME}:${BUILD_NUMBER}-${BUILD_DATE}
                '''               
                }   
              } 
              else {
                echo "The Image can't be pushed due to too many vulnerbilities"
                exit
              }                                    
            }
	        }
        }
      }
 
     stage('Asking for Deploy in prod') {
            //   when {
            //      equals(actual: env.gitlabBranch , expected: "prod")
            //  }
            when {
                branch 'main'
            }
             steps {
                 input "Do you want to Deploy in Production?"
             }
         }
 
     stage('Deploy') {
      when {
                branch 'main'
            }
       steps {    
          container('dind') {
            script {
             withCredentials([gitUsernamePassword(credentialsId: 'jenkins-demo', gitToolName: 'Default')]) {
             last_started = env.STAGE_NAME
               sh '''
               git clone -b $HELM_CHART_GIT_BRANCH $HELM_CHART_GIT_REPO_URL
               git config --global user.email $GIT_USER_EMAIL
               git config --global user.name $GIT_USER_NAME
               cd helm
               yq e -i '(."'${DOCKER_REPO_NAME}'".image.repository = "'${DOCKER_REPO_BASE_URL}'/'${DOCKER_REPO_NAME}'/'${DEPLOYMENT_STAGE}'")' env/${DEPLOYMENT_STAGE}/values.yaml
               yq e -i '(."'${DOCKER_REPO_NAME}'".image.tag = "'${BUILD_NUMBER}-${BUILD_DATE}'")' env/${DEPLOYMENT_STAGE}/values.yaml
               git add .
               git commit -m 'Docker Image version Update "'$JOB_NAME'"-"'$BUILD_NUMBER-$BUILD_DATE'"'
               git push origin $HELM_CHART_GIT_BRANCH
           '''
         } 
       }
     }
   }
     }
  }

      post {
        failure {
            slackSend message: 'Pipeline for ' + env.JOB_NAME + ' with Build Id - ' +  env.BUILD_ID + ' Failed at - ' +  env.last_started
        }

        success {
            slackSend message: 'Pipeline for ' + env.JOB_NAME + ' with Build Id - ' +  env.BUILD_ID + ' SUCCESSFUL'
        }
    }
}

