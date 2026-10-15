node {
    def buildStatus = 'SUCCESS'

    // Java 17 configuration for the entire pipeline
    withEnv([
        'JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64',
        'PATH+JAVA=/usr/lib/jvm/java-17-openjdk-amd64/bin'
    ]) {

        try {

            // Stage 1: Checkout source code
            stage('Checkout') {
                checkout scm
            }

            // Stage 2: Environment verification
            stage('Environment Check') {
                sh '''
                    set -eu

                    echo "Checking Java 17..."
                    java -version
                    javac -version

                    echo "Checking Maven Wrapper..."
                    test -f mvnw
                    chmod +x mvnw
                    ./mvnw -version

                    echo "Checking Gitleaks..."
                    gitleaks version
                '''
            }

            // Stage 3: Scan repository for secrets
            stage('Gitleaks Secret Scan') {
                sh '''
                    set -eu

                    echo "Starting Gitleaks secret scan..."

                    gitleaks git . \
                      --redact \
                      --exit-code 1

                    echo "Gitleaks scan completed successfully."
                '''
            }

            // Stage 4: Compile, test and package the application
            stage('Maven Build') {
                sh '''
                    set -eu

                    echo "Java used for Maven:"
                    java -version
                    javac -version

                    echo "Running Maven build and tests..."
                    ./mvnw -B clean package
                '''
            }

            // Stage 5: SonarQube code analysis
            stage('SonarQube Analysis') {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        set -eu

                        echo "Starting SonarQube analysis..."

                        ./mvnw -B \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:5.3.0.6276:sonar \
                          -Dsonar.projectKey=salary-api \
                          -Dsonar.host.url=http://10.0.18.41:9000

                        echo "SonarQube analysis completed."
                    '''
                }
            }

            // Stage 6: Wait for SonarQube Quality Gate result
            stage('Quality Gate') {
                timeout(time: 5, unit: 'MINUTES') {

                    def qualityGate = waitForQualityGate()

                    echo "SonarQube Quality Gate status: ${qualityGate.status}"

                    if (qualityGate.status != 'OK') {
                        error "SonarQube Quality Gate failed: ${qualityGate.status}"
                    }

                    echo "Quality Gate passed successfully."
                }
            }

        } catch (err) {

            buildStatus = 'FAILURE'
            throw err

        } finally {

            echo "Pipeline finished with status: ${buildStatus}"
        }
    }
}
