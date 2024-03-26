#!/usr/bin/env groovy

library identifier: 'jenkins-shared-library@master', retriever: modernSCM(
    [$class: 'GitSCMSource',
     remote: 'https://github.com/dhgiang85/jenkins-shared-library.git',
    //  credentialsId: 'gitlab-credentials'
    ]
)

pipeline {
    agent any
    tools {
        maven 'mymaven'
    }
//     environment {
//         IMAGE_NAME = 'dhgiant/demo-app:jma-2.0'
//     }
    stages {
       stage('Increase app version') {
              steps {
                 script {
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
//                     echo ${matcher}
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "dhgiant/demo-app:jma-${version}-${BUILD_NUMBER}"
                 }
              }
        }
        stage('build app') {
            steps {
               script {
                  echo 'building application jar...'
                  buildJar()
               }
            }
        }
        stage('build image') {
            steps {
                script {
                   echo 'building docker image...'
                   buildImage(env.IMAGE_NAME)
                   dockerLogin()
                   dockerPush(env.IMAGE_NAME)
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                   echo 'deploying docker image to EC2...'

                   def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME}"
                   def ec2Instance = "ec2-user@3.0.146.211"

                   sshagent(['ec2-server-key']) {
                       sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
                       sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                       sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
                   }
                }
            }
        }
        stage('commit updated version') {
            steps {
               script {
                withCredentials([usernameAndPassword(credentialsId:'github-user-repo', passwordVariable:'PASS', usernameVariable:'USER')])
                 sh 'git remote set-url origin https://$USER:$PASS@github.com/$USER/java-maven-app.git'
                 sh 'git add .'
                 sh 'git commit -m "CI: version bump"'
                 sh 'git push origin HEAD:jenkins-job'
               }
            }
      }
    }
}
