#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        DOCKER_REPO_SERVER = 'docker.io'
        DOCKER_REPO = 'tchamrss/java-maven-app'
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "docker build -t ${DOCKER_REPO}:${IMAGE_NAME} ."
                        sh 'echo $PASS | docker login -u $USER --password-stdin docker.io'
                        sh "docker push ${DOCKER_REPO}:${IMAGE_NAME}"
                    }
                }
            }
        }
        stage('deploy') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
                AWS_DEFAULT_REGION = 'eu-central-1'
                APP_NAME = 'java-maven-app'
            }

            steps {
                script {
                    echo 'deploying docker image...'

                    sh '''
                        set -e

                        echo "Checking AWS identity..."
                        aws sts get-caller-identity

                        echo "Creating EKS kubeconfig..."
                        aws eks update-kubeconfig \
                            --region eu-central-1 \
                            --name demo-cluster

                        echo "Checking Kubernetes connection..."
                        kubectl config current-context
                        kubectl get nodes

                        echo "Deploying application..."
                        envsubst < kubernetes/deployment.yaml | kubectl apply -f -

                        envsubst < kubernetes/service.yaml | kubectl apply -f -
                    '''
                }
            }
        }

        stage('commit version update') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'git-push-credential',
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]) {
                        sh '''
                            set +x
                            git add .

                            if ! git diff --cached --quiet; then
                                git -c user.name="Jenkins" \
                                    -c user.email="jenkins@localhost" \
                                    commit -m "ci: version bump"
                            fi

                            AUTH=$(printf '%s:%s' "$GIT_USER" "$GIT_TOKEN" | base64 | tr -d '\\n')
                            git -c http.extraheader="AUTHORIZATION: basic $AUTH" \
                                push origin HEAD:jenkins-jobs
                        '''
                    }
                }
            }
        }
    }
}
