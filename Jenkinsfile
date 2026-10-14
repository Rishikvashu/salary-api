node {
    def buildStatus = 'SUCCESS'

    try {
        stage('Checkout') {
            checkout scm
        }

        stage('Environment Check') {
            sh '''
                echo "Checking Java 17..."
                /usr/lib/jvm/java-17-openjdk-amd64/bin/java -version
                /usr/lib/jvm/java-17-openjdk-amd64/bin/javac -version

                echo "Checking Maven Wrapper..."
                test -f mvnw
                chmod +x mvnw

                echo "Checking Gitleaks..."
                gitleaks version
            '''
        }

        stage('Gitleaks Secret Scan') {
            sh '''
                gitleaks git . \
                  --redact \
                  --exit-code 1
            '''
        }

        stage('Maven Build') {
            withEnv([
                'JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64',
                'PATH+JAVA=/usr/lib/jvm/java-17-openjdk-amd64/bin'
            ]) {
                sh '''
                    echo "Java used for Maven:"
                    java -version
                    javac -version

                    ./mvnw -B clean package
                '''
            }
        }

        stage('SonarQube Analysis') {
            withEnv([
                'JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64',
                'PATH+JAVA=/usr/lib/jvm/java-17-openjdk-amd64/bin'
            ]) {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ./mvnw -B sonar:sonar \
                          -Dsonar.projectKey=salary-api \
                          -Dsonar.host.url=http://10.0.18.41:9000
                    '''
                }
            }
        }

        stage('Quality Gate') {
            timeout(time: 5, unit: 'MINUTES') {
                def qualityGate = waitForQualityGate()
                if (qualityGate.status != 'OK') {
                    error "SonarQube Quality Gate failed: ${qualityGate.status}"
                }
            }
        }

    } catch (err) {
        buildStatus = 'FAILURE'
        throw err
    } finally {
        echo "Pipeline finished with status: ${buildStatus}"
    }
}
