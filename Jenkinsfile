@Library('dynatrace@master') _

def tagMatchRules = [
  [
    meTypes: [
      [meType: 'SERVICE']
    ],
    tags : [
      [context: 'CONTEXTLESS', key: 'app', value: ''],
      [context: 'CONTEXTLESS', key: 'environment', value: 'staging']
    ]
  ]
]

pipeline {
  parameters {
    string(name: 'APP_NAME', defaultValue: '', description: 'The name of the service to deploy.', trim: true)
    string(name: 'TAG_STAGING', defaultValue: '', description: 'The image of the service to deploy.', trim: true)
    string(name: 'VERSION', defaultValue: '', description: 'The version of the service to deploy.', trim: true)
    string(name: 'DT_CUSTOM_PROP', defaultValue: '', description: 'Custom properties to be supplied to Dynatrace.', trim: true)
    choice(name: 'QUALITYGATE_PROVIDER', choices: ['Performance Signature Plugin','Keptn Quality Gates', 'None'], description: 'Select your evaluation provider. \'None\' will not evaluate', trim: true)
  }
  agent {
    label 'kubegit'
  }
  stages {
    stage('Update Deployment and Service specification') {
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'git-credentials-acm', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
            sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
            sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
            sh "cd k8s-deploy-staging/ && sed 's#value: \"DT_CUSTOM_PROP_PLACEHOLDER\".*#value: \"${env.DT_CUSTOM_PROP}\"#' ${env.APP_NAME}.yml > manifest-gen/${env.APP_NAME}.yml"
            sh "cd k8s-deploy-staging/ && sed -i 's#image: .*#image: ${env.TAG_STAGING}#' manifest-gen/${env.APP_NAME}.yml"
            sh "cd k8s-deploy-staging/ && git add manifest-gen/${env.APP_NAME}.yml && git commit -m 'Update ${env.APP_NAME} version ${env.VERSION}'"
            sh "cd k8s-deploy-staging/ && git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/k8s-deploy-staging"
            sh "rm -rf k8s-deploy-staging"
          }
        }
      }
    }
    stage('Deploy to staging namespace') {
      steps {
        checkout scm
        container('kubectl') {
          sh "kubectl -n staging apply -f manifest-gen/${env.APP_NAME}.yml"
        }
      }
    }
    // DO NOT uncomment until 06_04 Lab
    
    stage('DT Deploy Event') {
      steps {
        container("curl") {
          script {
            tagMatchRules[0].tags[0].value = "${env.APP_NAME}"
            def status = pushDynatraceDeploymentEvent (
              tagRule : tagMatchRules,
              customProperties : [
                [key: 'Jenkins Build Number', value: "${env.BUILD_ID}"],
                [key: 'Git commit', value: "${env.GIT_COMMIT}"],
                [key: 'Last commit by', value: "${env.GIT_COMMITTER_NAME}"],
                [key: 'Branch', value: "${env.GIT_BRANCH}"],
                [key: 'SCM', value: "${env.GIT_URL}"]
              ]
            )
          }
        }
      }
    }
    
    
    // DO NOT uncomment until 10_01 Lab
    
    stage('Staging Warm Up') {
      when {
        expression {
          return params.QUALITYGATE_PROVIDER == 'Performance Signature Plugin'
        }
      }
      steps {
        echo "Waiting for the service to start..."
        container('kubectl') {
          script {
            def status = waitForDeployment (
              deploymentName: "${env.APP_NAME}",
              environment: 'staging'
            )
            if(status !=0 ){
              currentBuild.result = 'FAILED'
              error "Deployment did not finish before timeout."
            }
          }
        }
        echo "Running one iteration with one VU to warm up service"  
        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: "jmeter/front-end_e2e_load.jmx",
              resultsDir: "e2eCheck_${env.APP_NAME}_warmup_${env.VERSION}_${BUILD_NUMBER}",
              serverUrl: "front-end.staging", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "e2eCheck_${BUILD_NUMBER}_warmup",
              funcValidation: false,
              avgRtValidation: 4000
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Warm up round in staging failed."
            }
          }
        }
        sleep 60
      }
    }

    stage('Run production ready e2e check in staging') {
      when {
        expression {
          return params.QUALITYGATE_PROVIDER == 'Performance Signature Plugin'
        }
      }
      steps {
        recordDynatraceSession (
          envId: 'Dynatrace Tenant',
          testCase: 'loadtest',
          tagMatchRules: tagMatchRules
        ) 
        {
          container('jmeter') {
            script {
              def status = executeJMeter ( 
                scriptName: "jmeter/front-end_e2e_load.jmx",
                resultsDir: "e2eCheck_${env.APP_NAME}_staging_${env.VERSION}_${BUILD_NUMBER}",
                serverUrl: "front-end.staging", 
                serverPort: 80,
                checkPath: '/health',
                vuCount: 10,
                loopCount: 5,
                LTN: "e2eCheck_${BUILD_NUMBER}",
                funcValidation: false,
                avgRtValidation: 4000
              )
              if (status != 0) {
                currentBuild.result = 'FAILED'
                error "Production ready e2e check in staging failed."
              }
            }
          }
        }
        //sleeping to allow data to arrive in Dynatrace
        sleep 60
        perfSigDynatraceReports(
          envId: 'Dynatrace Tenant', 
          nonFunctionalFailure: 2, 
          specFile: "monspec/e2e_perfsig.json"
        )
      }
    }
    stage('Keptn Quality Gate') {
      when {
        expression {
          return params.QUALITYGATE_PROVIDER == 'Keptn Quality Gates'
        }
      }
      steps {
        build job: "sockshop/{env.APP_NAME}.keptn/master"
      }
    }
    
  }
}
