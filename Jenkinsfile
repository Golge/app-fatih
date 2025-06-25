pipeline {
    agent any
    
    tools {
        maven 'maven'
        jdk 'jdk'
        dockerTool 'docker'
    }
    
    environment {
        HARBOR_REGISTRY = '34.147.56.126:30083'
        HARBOR_PROJECT = 'javdes'
        IMAGE_NAME = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/app-javdes"
        TAG = "${env.BUILD_NUMBER}"
        SONAR_PROJECT_KEY = 'app-javdes'
    }

    stages {

        stage('Tool Check') {
            steps {
                sh '''
                    echo "Java: $(which java)"
                    echo "Maven: $(which mvn)"
                    echo "Docker: $(which docker)"
                '''
            }
        }
        
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "✅ Code checked out successfully"
            }
        }
        
        stage('Environment Info') {
            steps {
                sh '''
                    echo "=== Build Information ==="
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Image Name: ${IMAGE_NAME}"
                    echo "Tag: ${TAG}"
                    echo "Harbor Registry: ${HARBOR_REGISTRY}"
                    echo "========================="
                    
                    echo "=== Tool Versions ==="
                    java -version
                    mvn -version
                    docker --version
                    echo "===================="
                '''
            }
        }
        
        stage('Build & Test') {
            steps {
                sh '''
                    mvn clean compile
                    mvn test
                    mvn package -DskipTests=true
                '''
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, allowEmptyArchive: true
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                sh '''
                    echo "Running SonarQube analysis in Docker..."
                    docker run --rm \
                        -v "${WORKSPACE}:/usr/src" \
                        -w /usr/src \
                        sonarsource/sonar-scanner-cli:latest \
                        sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=src/main/java \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.working.directory=.scannerwork || true
                    echo "✅ SonarQube analysis completed"
                '''
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t ${IMAGE_NAME}:${TAG} .
                    docker build -t ${IMAGE_NAME}:latest .
                    echo "✅ Docker image built successfully"
                '''
            }
        }
        
        stage('Push to Harbor') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-registry', 
                    usernameVariable: 'HARBOR_USER', 
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh '''
                        echo "Logging into Harbor registry..."
                        echo ${HARBOR_PASS} | docker login ${HARBOR_REGISTRY} -u ${HARBOR_USER} --password-stdin
                        
                        echo "Pushing Docker images..."
                        docker push ${IMAGE_NAME}:${TAG}
                        docker push ${IMAGE_NAME}:latest
                        
                        echo "✅ Images pushed successfully!"
                    '''
                }
            }
        }
        
        stage('Update Manifests') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token', 
                    usernameVariable: 'GIT_USERNAME', 
                    passwordVariable: 'GIT_PASSWORD'
                )]) {
                    sh '''
                        # Clean and clone manifests repo
                        rm -rf app-fatih-manifests
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Golge/app-fatih-manifests.git
                        cd app-fatih-manifests
                        
                        # Update deployment image
                        if [ -f "k8s-manifests/app/base/deployment.yaml" ]; then
                            sed -i "s|image:.*app-javdes:.*|image: ${IMAGE_NAME}:${TAG}|g" k8s-manifests/app/base/deployment.yaml
                            
                            # Commit and push changes
                            git config user.email "fatihgumush@gmail.com"
                            git config user.name "Golge"
                            git add .
                            git commit -m "Update app-javdes image to ${TAG}" || echo "No changes to commit"
                            git push origin main
                            
                            echo "✅ Manifests updated successfully!"
                        else
                            echo "❌ Deployment manifest not found"
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                docker system prune -f || true
                rm -rf app-fatih-manifests || true
            '''
        }
        
        success {
            echo "🎉 Build successful! Image: ${IMAGE_NAME}:${TAG}"
        }
        
        failure {
            echo "💥 Build failed at stage: ${env.STAGE_NAME}"
        }
    }
}
