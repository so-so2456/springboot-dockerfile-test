// 리팩토링 필요
pipeline {
    agent any

    parameters {
        string(name : 'application_name', defaultValue : 'demo', description : 'application_name and used to manifest. this must be unique')
        string(name : 'github_link', defaultValue : 'https://github.com/so-so2456/springboot-dockerfile-test.git', description : 'github link of user')
        string(name : 'github_branch', defaultValue : 'main', description : 'github branch of user')
    }

    environment {
        // Argo manifest가 있는 github organization
        ORG_NAME = "so-so2456"
        TEMPLATE_URL = "https://github.com/so-so2456/argocd-manifest-template.git"
        GITHUB_USERNAME = "so-so2456"
        ARGOCD_NAMESPACE = "argo"
        NEXUS_IP = "localhost:5443"
        NEXUS_ID = "admin"
        NEXUS_PW = "123"
    }

    stages {
        stage('checkout') {
            steps {
                git branch: "${params.github_branch}", poll: false, url: "${params.github_link}"
            }
        }
        stage('Build Project') {
            steps {
                sh './mvnw clean package'
            }
        }
        stage("Build a Image") {
            // 도커 이미지, 태그 네임 정책 필요
            steps {
                sh "docker build -t ${env.NEXUS_IP}/demo-image:${BUILD_NUMBER} ."
            }
		}
        stage("Push the Image in Nexus") {
            // 넥서스에 푸쉬
            steps {
                sh "docker push ${env.NEXUS_IP}/demo-image:${BUILD_NUMBER}"
            }
		}
    }
}