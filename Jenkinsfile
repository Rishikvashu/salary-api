node {
    def buildStatus = 'SUCCESS'

    withEnv([
        'JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64',
        'PATH+JAVA=/usr/lib/jvm/java-17-openjdk-amd64/bin'
    ]) {
        try {

            stage('Checkout') {
                checkout scm
            }

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

            stage('Gitleaks Secret Scan') {
                sh '''
                    set -eu
                    echo "Starting Gitleaks secret scan..."
                    gitleaks git . --redact --exit-code 1
                    echo "Gitleaks scan completed successfully."
                '''
            }

            stage('Maven Build') {
                sh '''
                    set -eu
                    echo "Running Maven build and tests..."
                    ./mvnw -B clean package
                '''
            }

            stage('SonarQube Analysis') {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        set -eu
                        echo "Starting SonarQube analysis..."

                        ./mvnw -B \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:5.3.0.6276:sonar \
                          -Dsonar.projectKey=salary-api

                        echo "SonarQube analysis completed."
                    '''
                }
            }

            stage('Quality Gate') {
                timeout(time: 5, unit: 'MINUTES') {
                    def qg = waitForQualityGate()
                    echo "SonarQube Quality Gate status: ${qg.status}"
                    if (qg.status != 'OK') {
                        error "SonarQube Quality Gate failed: ${qg.status}"
                    }
                }
            }

        } catch (err) {
            buildStatus = 'FAILURE'
            currentBuild.result = 'FAILURE'
            throw err
        } finally {
            echo "Pipeline finished with status: ${buildStatus}"
        }
    }
}
