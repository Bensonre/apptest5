pipeline {
    agent any

    tools {
        nodejs 'nodejs'
    }

    stages {
        stage('Pull') {
            steps {
                git url: "${githubURL}", branch: "${gitBranch}"
            }
        }
        stage('Build') {
            steps {
                sh 'npm install'
            }
        }
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        stage('Deploy') {
            steps {
                script {
                    docker.withTool('docker') {
                        repoId = "${projectName}"
                        image = docker.build(repoId)
                        docker.withRegistry("${ContainerRegistryURL}", "${ContainerRegistryCredId}") {
                            image.push()
                        }
                    }
                }
            }
        }
        stage('Helm Tests') {
            steps {
                withKubeConfig([credentialsId: "${kubectlCreds}", serverUrl: "${kubectlServer}"]) {
                    sh "helm install helmtest Chart -n helm-test"
                    sleep(10);
                    sh 'helm test helmtest -n helm-test'
                }
            }
        }
        stage('Upload Helm Chart')
        {
            environment{
                chartname = sh (script: "helm package Chart | sed 's/^[^:]*: //g'", returnStdout:true).trim()
            }
            steps {
                sh "curl --data-binary \"@${env.chartname}\" ${helmChartRepoURL}"
            }
        }
    }
    post {
        success {
            slackSend(color: 'good', channel: "${slackChannel}", message: "Maven project '${projectName}' [${gitBranch}] has passed all tests and was successfully built and pushed to the CR.")
        }
        failure {
            slackSend(color: 'danger', channel: "${slackChannel}", message: "Maven project '${projectName}' [${gitBranch}] has failed to complete pipeline.")
        }
        always {
            withKubeConfig([credentialsId: "${kubectlCreds}", serverUrl: "${kubectlServer}"]) {
                sh 'helm uninstall helmtest -n helm-test'
            }
        }
    }
}