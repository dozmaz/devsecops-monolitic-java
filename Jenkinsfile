pipeline {
    agent any

    // -------------------------------------------------
    // Herramientas configuradas en Jenkins → Global Tool Configuration
    // -------------------------------------------------
    tools {
        jdk    'JDK-21'      // nombre del JDK registrado en Jenkins
        gradle 'Gradle-8'    // nombre del Gradle registrado en Jenkins
    }

    // -------------------------------------------------
    // Variables de entorno (credenciales en Jenkins Credentials Store)
    // -------------------------------------------------
    environment {
        SONAR_PROJECT_KEY  = 'spring-boot-complete'
        SONAR_PROJECT_NAME = 'Spring Boot Complete'
        ARTIFACTORY_REPO   = 'libs-release-local'
        GRADLE_USER_HOME = '/var/jenkins_home/.gradle'
    }

    options {
        // Skip default 10-minute checkout so we can use our custom timeout
        skipDefaultCheckout(true)
    }

    stages {
        stage('Checkout') {
            steps {
                // 1. Borrar todos los archivos ocultos y visibles del directorio actual
                deleteDir() 
                
                // 2. Ejecutar la clonación omitiendo el certificado SSL
                sh 'git -c http.sslVerify=false clone https://github.com/dozmaz/devsecops-monolitic-java.git .'
            }
        }

        // --- STAGE 1: Build ---------------------------
        stage('Build') {
            steps {
                echo '=== STAGE: Build ==='
                sh 'chmod +x ./gradlew'
                sh './gradlew clean assemble --no-daemon'
            }
            post {
                failure { error 'Build fallido. Revisa los errores de compilacion.' }
            }
        }

        // --- STAGE 2: Testing (JUnit + JaCoCo) --------
        stage('Testing') {
            steps {
                echo '=== STAGE: Testing (JUnit + JaCoCo) ==='
                sh './gradlew test jacocoTestReport --no-daemon'
            }
            post {
                always {
                    junit '**/build/test-results/test/*.xml'
                    jacoco(
                        execPattern:         '**/build/jacoco/*.exec',
                        classPattern:        '**/build/classes/java/main',
                        sourcePattern:       '**/src/main/java',
                        exclusionPattern:    '**/test/**',
                        minimumLineCoverage: '60'
                    )
                }
                failure { error 'Tests fallidos. Revisa los reportes JUnit/JaCoCo.' }
            }
        }

        // --- STAGE 3: SonarQube -----------------------
        stage('Sonar') {
            steps {
                echo '=== STAGE: SonarQube Analysis ==='
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ./gradlew sonar --no-daemon \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.projectName='${SONAR_PROJECT_NAME}' \
                          -Dsonar.java.source=17 \
                          -Dsonar.coverage.jacoco.xmlReportPaths=build/reports/jacoco/test/jacocoTestReport.xml
                    """
                }
            }
            post {
                always {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        // --- STAGE 4: Artefacto → Artifactory ---------
        stage('Generar artefacto y almacenar en Artifactory') {
            steps {
                echo '=== STAGE: Publish to Artifactory ==='
                sh './gradlew bootJar --no-daemon'

                rtUpload(
                    serverId: 'Artifactory',
                    spec: """{
                        "files": [{
                            "pattern": "build/libs/*.jar",
                            "target":  "${ARTIFACTORY_REPO}/com/example/spring-boot-complete/${BUILD_NUMBER}/"
                        }]
                    }"""
                )

                rtPublishBuildInfo(
                    serverId: 'Artifactory'
                )
            }
            post {
                failure { error 'Error al publicar en Artifactory.' }
            }
        }
    }

    post {
        success { echo "Pipeline completado exitosamente. Build #${BUILD_NUMBER}" }
        failure { echo "Pipeline fallido. Build #${BUILD_NUMBER}" }
        always  { cleanWs() }
    }
}
