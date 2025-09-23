pipeline {
    agent none  // Use this so that we can define agents for individual stages
    
    environment {
        NETLIFY_SITE_ID = '2bddd7ef-0d9f-4e84-b360-9e983f4337a1'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
        stage('Initialize Docker Environment') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true  // Reuse the node environment across stages
                }
            }
            steps {
                // Install global dependencies once and use them throughout the pipeline
                sh '''
                    npm install -g netlify-cli serve
                    node_modules/.bin/netlify --version
                    node_modules/.bin/serve --version
                '''
            }
        }

        stage('Build') {
            agent { docker { image 'node:18-alpine' reuseNode true } }
            steps {
                sh '''
                    npm ci
                    npm run build
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent { docker { image 'node:18-alpine' reuseNode true } }
                    steps {
                        sh 'npm test'
                    }
                    post {
                        always {
                            junit 'test-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent { docker { image 'mcr.microsoft.com/playwright:v1.39.0-jammy' reuseNode true } }
                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report'])
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            agent { docker { image 'node:18-alpine' reuseNode true } }
            steps {
                sh '''
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify deploy --dir=build --prod
                '''
            }
        }
    }
}
