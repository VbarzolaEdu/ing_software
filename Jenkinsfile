#!/usr/bin/env groovy

node {
    stage('checkout') {
        checkout scm
    }

    gitlabCommitStatus('build') {
        docker.image('jhipster/jhipster:v8.11.0').inside('-u jhipster -e MAVEN_OPTS="-Duser.home=./"') {
            stage('check java') {
                sh "java -version"
            }

            stage('clean') {
                sh "chmod +x mvnw"
                sh "./mvnw -ntp clean -P-webapp"
            }
            stage('nohttp') {
                sh "./mvnw -ntp checkstyle:check"
            }

            stage('install tools') {
                sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:install-node-and-npm@install-node-and-npm"
            }

            stage('npm install') {
                sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:npm"
            }
            stage('Install Snyk CLI') {
               sh '''
                   curl -Lo ./snyk $(curl -s https://api.github.com/repos/snyk/snyk/releases/latest | grep "browser_download_url.*snyk-linux" | cut -d ':' -f 2,3 | tr -d \" | tr -d ' ')
                   chmod +x snyk
               '''
            }
            stage('Snyk test') {
               sh './snyk test --all-projects'
            }
            stage('Snyk monitor') {
               sh './snyk monitor --all-projects'
            }
            stage('backend tests') {
                try {
                    sh "./mvnw -ntp verify -P-webapp"
                } catch(err) {
                    throw err
                } finally {
                    junit '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
                }
            }

            stage('frontend tests') {
                try {
                   sh "npm install"
                   sh "npm test"
                } catch(err) {
                    throw err
                } finally {
                    junit '**/target/test-results/TESTS-results-jest.xml'
                }
            }

            stage('package and deploy') {
                sh "./mvnw -ntp com.heroku.sdk:heroku-maven-plugin:3.0.7:deploy -DskipTests -Pprod -Dheroku.buildpacks=heroku/jvm -Dheroku.appName="
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
            stage('quality analysis') {
                withSonarQubeEnv('') {
                    sh "./mvnw -ntp initialize sonar:sonar"
                }
            }
        }

        def dockerImage
        stage('publish docker') {
            withCredentials([usernamePassword(credentialsId: 'dockerhub-login', passwordVariable: 'DOCKER_REGISTRY_PWD', usernameVariable: 'DOCKER_REGISTRY_USER')]) {
                sh "./mvnw -ntp jib:build"
            }
        }
    }
}
