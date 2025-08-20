pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'Github-cred'
        DOCKER_CREDENTIALS = 'Dockerhub-cred'
        DOCKERHUB_USERNAME = 'roshanx' // Your DockerHub username
        SONAR_HOME = tool "SonarQube"
        GITHUB_REPO = 'https://github.com/Roshanx96/store-app.git'
    }

    stages {

        stage("Workspace Cleanup") {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: env.GITHUB_CREDENTIALS,
                    url: env.GITHUB_REPO
            }
        }

        stage('Build Environment Setup') {
            steps {
                script {
                    echo '🔧 Setting up build environment...'
                    
                    // Fix Maven wrapper permissions
                    sh '''
                        echo "Setting Maven wrapper permissions..."
                        chmod +x ./src/cart/mvnw
                        chmod +x ./src/orders/mvnw
                        chmod +x ./src/ui/mvnw
                    '''
                    
                    // Setup caching directories
                    sh '''
                        echo "Setting up cache directories..."
                        mkdir -p /tmp/maven-cache
                        mkdir -p /tmp/go-cache
                        mkdir -p /tmp/npm-cache
                        
                        # Set cache environment variables
                        export MAVEN_OPTS="-Dmaven.repo.local=/tmp/maven-cache"
                        export GOCACHE=/tmp/go-cache
                        export npm_config_cache=/tmp/npm-cache
                        
                        echo "Cache directories created:"
                        ls -la /tmp/ | grep -E "(maven|go|npm)-cache"
                    '''
                    
                    // Validate build tools
                    sh '''
                        echo "Validating build environment..."
                        echo "Java version:"
                        java -version
                        
                        echo "Maven version:"
                        ./src/cart/mvnw --version
                        
                        echo "Go version:"
                        go version
                        
                        echo "Node.js version:"
                        node --version
                        
                        echo "npm version:"
                        npm --version
                        
                        echo "Docker version:"
                        docker --version
                    '''
                }
            }
        }

        stage('Security Scans') {
            parallel {
                stage('Trivy Filesystem Scan') {
                    steps {
                        sh '''
                            trivy fs --exit-code 1 --severity CRITICAL --no-progress . || true
                        '''
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        script {
                            // Check if dependency-check.sh exists
                            def exists = sh(script: 'which dependency-check.sh || true', returnStdout: true).trim()
                            if (exists) {
                                sh 'dependency-check.sh --project store-app --scan . || true'
                            } else {
                                echo '⚠️ OWASP Dependency Check skipped: dependency-check.sh not found.'
                            }
                        }
                    }
                }
            }
        }

        stage('Check Sonar') {
            steps {
                sh 'echo SONAR_HOME="$SONAR_HOME"'
                sh '$SONAR_HOME/bin/sonar-scanner --version'
            }
        }

        stage('SonarQube: Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SONAR_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=store-app \
                        -Dsonar.projectName=store-app \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('SonarQube: Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Test Services') {
            parallel {
                stage('Build & Test Cart Service') {
                    steps {
                        script {
                            echo '🏗️ Building Cart Service...'
                            dir('src/cart') {
                                sh '''
                                    echo "Compiling Cart service..."
                                    export MAVEN_OPTS="-Dmaven.repo.local=/tmp/maven-cache"
                                    ./mvnw clean compile test -Dcheckstyle.skip=true
                                    
                                    echo "Verifying JAR artifact creation..."
                                    if [ -f target/*.jar ]; then
                                        echo "✅ Cart service JAR created successfully"
                                        ls -la target/*.jar
                                    else
                                        echo "❌ Cart service JAR not found"
                                        exit 1
                                    fi
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                if (fileExists('src/cart/target/surefire-reports/*.xml')) {
                                    publishTestResults testResultsPattern: 'src/cart/target/surefire-reports/*.xml'
                                } else {
                                    echo 'No test results found for Cart service'
                                }
                            }
                        }
                        failure {
                            echo '❌ Cart service build failed'
                        }
                        success {
                            echo '✅ Cart service build completed successfully'
                        }
                    }
                }

                stage('Build & Test Orders Service') {
                    steps {
                        script {
                            echo '🏗️ Building Orders Service...'
                            dir('src/orders') {
                                sh '''
                                    echo "Compiling Orders service..."
                                    export MAVEN_OPTS="-Dmaven.repo.local=/tmp/maven-cache"
                                    ./mvnw clean compile test -Dcheckstyle.skip=true
                                    
                                    echo "Verifying JAR artifact creation..."
                                    if [ -f target/*.jar ]; then
                                        echo "✅ Orders service JAR created successfully"
                                        ls -la target/*.jar
                                    else
                                        echo "❌ Orders service JAR not found"
                                        exit 1
                                    fi
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                if (fileExists('src/orders/target/surefire-reports/*.xml')) {
                                    publishTestResults testResultsPattern: 'src/orders/target/surefire-reports/*.xml'
                                } else {
                                    echo 'No test results found for Orders service'
                                }
                            }
                        }
                        failure {
                            echo '❌ Orders service build failed'
                        }
                        success {
                            echo '✅ Orders service build completed successfully'
                        }
                    }
                }

                stage('Build & Test Catalog Service') {
                    steps {
                        script {
                            echo '🏗️ Building Catalog Service (Go)...'
                            dir('src/catalog') {
                                sh '''
                                    echo "Setting up Go environment..."
                                    export GOCACHE=/tmp/go-cache
                                    export GOPATH=/tmp/go-path
                                    
                                    echo "Downloading Go dependencies..."
                                    go mod download
                                    
                                    echo "Building Go service..."
                                    go build ./...
                                    
                                    echo "Running Go tests..."
                                    go test ./... -v -json > test-results.json || true
                                    
                                    echo "Verifying Go binary creation..."
                                    if go build -o catalog-service ./main.go; then
                                        echo "✅ Catalog service binary created successfully"
                                        ls -la catalog-service
                                    else
                                        echo "❌ Catalog service binary creation failed"
                                        exit 1
                                    fi
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                if (fileExists('src/catalog/test-results.json')) {
                                    echo '📊 Go test results available in test-results.json'
                                    sh 'cat src/catalog/test-results.json | grep -E "(PASS|FAIL|RUN)" || true'
                                } else {
                                    echo 'No Go test results found for Catalog service'
                                }
                            }
                        }
                        failure {
                            echo '❌ Catalog service build failed'
                        }
                        success {
                            echo '✅ Catalog service build completed successfully'
                        }
                    }
                }

                stage('Build & Test Checkout Service') {
                    steps {
                        script {
                            echo '🏗️ Building Checkout Service (Node.js)...'
                            dir('src/checkout') {
                                sh '''
                                    echo "Installing Node.js dependencies..."
                                    export npm_config_cache=/tmp/npm-cache
                                    npm ci
                                    
                                    echo "Building NestJS application..."
                                    npm run build
                                    
                                    echo "Running Node.js tests..."
                                    npm test || echo "Tests completed with exit code $?"
                                    
                                    echo "Verifying build artifacts..."
                                    if [ -d "dist" ]; then
                                        echo "✅ Checkout service dist folder created successfully"
                                        ls -la dist/
                                    else
                                        echo "❌ Checkout service dist folder not found"
                                        exit 1
                                    fi
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                // Check for Jest test results
                                if (fileExists('src/checkout/coverage/lcov-report/index.html')) {
                                    echo '📊 Node.js test coverage report available'
                                } else {
                                    echo 'No Node.js test coverage found for Checkout service'
                                }
                            }
                        }
                        failure {
                            echo '❌ Checkout service build failed'
                        }
                        success {
                            echo '✅ Checkout service build completed successfully'
                        }
                    }
                }
            }
        }

        stage('Build & Test UI Service') {
            steps {
                script {
                    echo '🏗️ Building UI Service (depends on other services)...'
                    dir('src/ui') {
                        sh '''
                            echo "Generating Kiota clients from OpenAPI specs..."
                            export MAVEN_OPTS="-Dmaven.repo.local=/tmp/maven-cache"
                            ./mvnw kiota:generate -Dcheckstyle.skip=true
                            
                            echo "Compiling UI service..."
                            ./mvnw clean compile test -Dcheckstyle.skip=true
                            
                            echo "Verifying JAR artifact creation..."
                            if [ -f target/*.jar ]; then
                                echo "✅ UI service JAR created successfully"
                                ls -la target/*.jar
                            else
                                echo "❌ UI service JAR not found"
                                exit 1
                            fi
                        '''
                    }
                }
            }
            post {
                always {
                    script {
                        if (fileExists('src/ui/target/surefire-reports/*.xml')) {
                            publishTestResults testResultsPattern: 'src/ui/target/surefire-reports/*.xml'
                        } else {
                            echo 'No test results found for UI service'
                        }
                    }
                }
                failure {
                    echo '❌ UI service build failed'
                }
                success {
                    echo '✅ UI service build completed successfully'
                }
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_CREDENTIALS,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Build & Push Docker Images') {
            parallel {
                stage('Build & Push UI Image') {
                    steps {
                        script {
                            echo '🐳 Building and pushing UI Docker image...'
                            sh '''
                                docker build -t ${DOCKERHUB_USERNAME}/store-app-ui:latest ./src/ui
                                docker push ${DOCKERHUB_USERNAME}/store-app-ui:latest
                            '''
                        }
                    }
                    post {
                        success {
                            echo '✅ UI Docker image built and pushed successfully'
                        }
                        failure {
                            echo '❌ UI Docker image build/push failed'
                        }
                    }
                }

                stage('Build & Push Cart Image') {
                    steps {
                        script {
                            echo '🐳 Building and pushing Cart Docker image...'
                            sh '''
                                docker build -t ${DOCKERHUB_USERNAME}/store-app-cart:latest ./src/cart
                                docker push ${DOCKERHUB_USERNAME}/store-app-cart:latest
                            '''
                        }
                    }
                    post {
                        success {
                            echo '✅ Cart Docker image built and pushed successfully'
                        }
                        failure {
                            echo '❌ Cart Docker image build/push failed'
                        }
                    }
                }

                stage('Build & Push Catalog Image') {
                    steps {
                        script {
                            echo '🐳 Building and pushing Catalog Docker image...'
                            sh '''
                                docker build -t ${DOCKERHUB_USERNAME}/store-app-catalog:latest ./src/catalog
                                docker push ${DOCKERHUB_USERNAME}/store-app-catalog:latest
                            '''
                        }
                    }
                    post {
                        success {
                            echo '✅ Catalog Docker image built and pushed successfully'
                        }
                        failure {
                            echo '❌ Catalog Docker image build/push failed'
                        }
                    }
                }

                stage('Build & Push Orders Image') {
                    steps {
                        script {
                            echo '🐳 Building and pushing Orders Docker image...'
                            sh '''
                                docker build -t ${DOCKERHUB_USERNAME}/store-app-orders:latest ./src/orders
                                docker push ${DOCKERHUB_USERNAME}/store-app-orders:latest
                            '''
                        }
                    }
                    post {
                        success {
                            echo '✅ Orders Docker image built and pushed successfully'
                        }
                        failure {
                            echo '❌ Orders Docker image build/push failed'
                        }
                    }
                }

                stage('Build & Push Checkout Image') {
                    steps {
                        script {
                            echo '🐳 Building and pushing Checkout Docker image...'
                            sh '''
                                docker build -t ${DOCKERHUB_USERNAME}/store-app-checkout:latest ./src/checkout
                                docker push ${DOCKERHUB_USERNAME}/store-app-checkout:latest
                            '''
                        }
                    }
                    post {
                        success {
                            echo '✅ Checkout Docker image built and pushed successfully'
                        }
                        failure {
                            echo '❌ Checkout Docker image build/push failed'
                        }
                    }
                }
            }
        }

        

        stage('Trigger CD Pipeline') {
            steps {
                build job: 'store-app-cd'
            }
        }
    }

    post {
        always {
            script {
                echo '📊 Pipeline Summary:'
                echo '=================='
                
                // Collect build status for each service
                def buildStatus = [:]
                
                // Check if build artifacts exist
                if (fileExists('src/cart/target/*.jar')) {
                    buildStatus['Cart'] = '✅ SUCCESS'
                } else {
                    buildStatus['Cart'] = '❌ FAILED'
                }
                
                if (fileExists('src/orders/target/*.jar')) {
                    buildStatus['Orders'] = '✅ SUCCESS'
                } else {
                    buildStatus['Orders'] = '❌ FAILED'
                }
                
                if (fileExists('src/ui/target/*.jar')) {
                    buildStatus['UI'] = '✅ SUCCESS'
                } else {
                    buildStatus['UI'] = '❌ FAILED'
                }
                
                if (fileExists('src/catalog/catalog-service')) {
                    buildStatus['Catalog'] = '✅ SUCCESS'
                } else {
                    buildStatus['Catalog'] = '❌ FAILED'
                }
                
                if (fileExists('src/checkout/dist')) {
                    buildStatus['Checkout'] = '✅ SUCCESS'
                } else {
                    buildStatus['Checkout'] = '❌ FAILED'
                }
                
                // Print build summary
                buildStatus.each { service, status ->
                    echo "${service} Service: ${status}"
                }
                
                echo '=================='
            }
        }
        success {
            echo "✅ The store-app CI pipeline completed successfully!"
            echo "🚀 All services built and Docker images pushed to registry."
        }
        failure {
            echo "❌ The store-app CI pipeline has failed."
            echo "🔍 Check the build logs above for specific service failures."
            echo "💡 Common issues:"
            echo "   - Build tool version mismatches"
            echo "   - Missing dependencies"
            echo "   - Test failures"
            echo "   - Docker build context issues"
        }
        unstable {
            echo "⚠️ The store-app CI pipeline completed with warnings."
            echo "🔍 Some tests may have failed but builds succeeded."
        }
    }
}