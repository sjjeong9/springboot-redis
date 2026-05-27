pipeline {
    agent any

    stages {
        stage('Git Clone') {
            steps {
                checkout scm
            }
        }

        stage('Gradle Build') {
            steps {
                sh 'chmod +x ./gradlew'
                sh './gradlew build -x test --no-daemon'
            }
        }

        stage('Docker Build & Deploy') {
            steps {
                // -p 옵션으로 프로젝트 이름 고정 → Jenkins 재배포 시 기존 컨테이너 재사용
                sh 'docker compose -p springboot-redis up --build -d'
            }
        }

        stage('Health Check') {
            steps {
                sh 'sleep 15'
                sh 'curl -f http://localhost/'
            }
        }
    }

    post {
        success {
            echo '배포 성공'
        }
        failure {
            echo '배포 실패 — 콘솔 로그 확인'
        }
    }
}
