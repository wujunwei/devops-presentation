+++
title = "devops"
description = "A Hugo theme for creating Reveal.js presentations"
outputs = ["Reveal"]
[reveal_hugo]
theme="sky"
margin = 0.02
transition = "slide"
transition_speed = "slow"
[logo]
src = "favicon.ico"
+++

# a story about devops

long long ago ...

---

## jenkins file
```c

pipeline {
  agent {
       node {
            label 'nodejs'
         }
    }


    environment {
        DOCKER_CREDENTIAL_ID = 'harbor-id'
        GITLAB_CREDENTIAL_ID = 'gitlab-id'
        KUBECONFIG_CREDENTIAL_ID = 'local-kubeconfig'
        REGISTRY = 'harbor.imeete.com'
        DOCKERHUB_NAMESPACE = 'kbs-doc'
        GIT_ACCOUNT = 'wujunwei'
        APP_NAME = 'kbs-doc'
        SONAR_CREDENTIAL_ID = 'sonar-token'
        KUBECONFIG_ONLINE_ID = 'kubeconfig-online'
    }

    stages {
        stage ('checkout scm') {
            steps {
                checkout(scm)
                sh 'git submodule update --init --recursive'
            }
        }

        stage ('build & push') {
            steps {
             container ('nodejs') {
                    sh 'npm config set registry https://registry.npm.taobao.org && npm install'
                    sh 'npm install -D --save postcss'
              }
              container ('hugo') {
                    sh 'HUGO_ENV="production" hugo --gc'
              }
              container ('nodejs') {
                sh 'docker build -f ./Dockerfile -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER .'
              }
            }
        }

        stage('push latest'){
           when{
             branch 'master'
           }
           steps{
           container ('nodejs') {
               withCredentials([usernamePassword(passwordVariable : 'DOCKER_PASSWORD' ,usernameVariable : 'DOCKER_USERNAME' ,credentialsId : "$DOCKER_CREDENTIAL_ID" ,)]) {
                    sh """
                        echo \'$DOCKER_PASSWORD\' | docker login $REGISTRY -u \'$DOCKER_USERNAME\' --password-stdin
                        """
                    sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER'
                    sh 'docker tag  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest '
                    sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest'
                  }
             }
           }
        }
        stage('deploy to production') {
           steps {
             kubernetesDeploy(configs: 'deploy/test/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
             input(id: 'deploy-to-production', message: 'deploy to production?')
             kubernetesDeploy(configs: 'deploy/online/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_ONLINE_ID")
           }
        }
    }
}

```
[Start over](/#/1)