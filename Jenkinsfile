#!/usr/bin/env groovy

node {
    stage('Checkout') {
        checkout scm
    }

    docker.image('jhipster/jhipster:v8.11.0').inside('-u jhipster -e MAVEN_OPTS="-Duser.home=./"') {

        stage('Check Java') {
            sh "java -version"
        }

        stage('Clean') {
            sh "chmod +x mvnw"
            sh "./mvnw -ntp clean -P-webapp"
        }

        stage('Code Style Check') {
            sh "./mvnw -ntp checkstyle:check"
        }

        stage('Install Tools') {
            sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:install-node-and-npm@install-node-and-npm"
        }

        stage('NPM Install') {
            sh "./mvnw -ntp com.github.eirslett:frontend-maven-plugin:npm"
        }

        // 🔹 Snyk Security Scans
        stage('Install Snyk CLI') {
            sh '''
                curl -Lo ./snyk https://static.snyk.io/cli/latest/snyk-linux
                chmod +x snyk
            '''
        }

        stage('Snyk Test') {
            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                sh """
                    ./snyk auth \$SNYK_TOKEN
                    ./snyk test --all-projects || true
                """
            }
        }

        stage('Snyk Monitor') {
            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                sh """
                    ./snyk auth \$SNYK_TOKEN
                    ./snyk monitor --all-projects
                """
            }
        }

        // 🔹 Tests
        stage('Backend Tests') {
            try {
                sh "./mvnw -ntp verify -P-webapp"
            } catch (err) {
                throw err
            } finally {
                junit '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml'
            }
        }

        stage('Frontend Tests') {
            try {
                sh "npm install"
                sh "npm test"
            } catch (err) {
                throw err
            } finally {
                junit '**/target/test-results/TESTS-results-jest.xml'
            }
        }

        // 🔹 Quality Analysis
        // stage('Quality Analysis') {
        //     withSonarQubeEnv('') {
        //         sh "./mvnw -ntp initialize sonar:sonar"
        //     }
        // }

        // 🔹 Package and Deploy to Docker Hub
        stage('Package') {
            sh "./mvnw -ntp clean package -Pprod -DskipTests"
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }

        stage('Deploy to DockerHub') {
            withCredentials([usernamePassword(credentialsId: 'dockerhub-login', passwordVariable: 'DOCKER_REGISTRY_PWD', usernameVariable: 'DOCKER_REGISTRY_USER')]) {
                sh """
                    echo '🚀 Publicando imagen en Docker Hub...'
                    ./mvnw -ntp jib:build
                """
            }
        }
    }
}
