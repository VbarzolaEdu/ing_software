#!/usr/bin/env groovy

node {
    stage('checkout') {
        checkout scm
    }

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

        // 🔹 Snyk authentication and test
        stage('Install Snyk CLI') {
            sh '''
                curl -Lo ./snyk https://static.snyk.io/cli/latest/snyk-linux
                chmod +x snyk
            '''
        }

        stage('Snyk test') {
            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                sh """
                    ./snyk auth \$SNYK_TOKEN
                    ./snyk test --all-projects
                """
            }
        }

        stage('Snyk monitor') {
            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                sh """
                    ./snyk auth \$SNYK_TOKEN
                    ./snyk monitor --all-projects
                """
            }
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

    // 🔹 Docker publish stage
    stage('publish docker') {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-login', passwordVariable: 'DOCKER_REGISTRY_PWD', usernameVariable: 'DOCKER_REGISTRY_USER')]) {
            sh "./mvnw -ntp jib:build"
        }
    }
}
