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
        ARGOCD_URL = "localhost:8083"
        NEXUS_IP = "localhost:5443"
        NEXUS_ID = "admin"
        NEXUS_PW = "123"
        DOCKER_USERNAME = "homecoder"
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
                sh "docker build -t ${env.DOCKER_USERNAME}/demo-image:${BUILD_NUMBER} ."
            }
		}
        stage("Push the Image in Dockerhub") {
            // 빌드한 이미지를 개인 도커허브에 푸쉬
            steps {
                sh "docker push ${env.DOCKER_USERNAME}/demo-image:${BUILD_NUMBER}"
            }
		}
        stage("Check manifest repo exist & Create manifest repo if not exist") {
            steps {
                script {
                    // repo가 있는지 확인하고 없으면 repo 생성
                    def argo_manifest_repo = "https://github.com/${env.GITHUB_USERNAME}/${params.application_name}-manifest"
                    def check_repo_response = sh(
                        returnStdout: true,
                        script: "curl -w \'%{http_code}\' -o /dev/null " + argo_manifest_repo
                    )

                    if (check_repo_response == "404") {
                        println "argo_manifest_repo(${argo_manifest_repo}) is not exist. create manifest repo."
                        // argo manifest repo가 없으면 repo를 생성하고 초기화
                        withCredentials([string(credentialsId: 'github_token', variable: 'TOKEN')]) {
                        // manifest repo 생성
                        def github_create_repo_api = "https://api.github.com/user/repos "
                        def github_auth = "-H \"Authorization: Bearer ${TOKEN}\" "
                        def github_create_repo_body = "-d '{\"name\": \"${params.application_name}-manifest\"}'"

                        def create_repo_response = sh(
                            returnStdout: true,
                            script: "curl -w \'%{http_code}\' -o /dev/null "  + github_auth + github_create_repo_api + github_create_repo_body
                        )
                        if (create_repo_response != "201"){
                            error("Failed to create repository, response code: ${response}")
                        }

                        // manifest repo에 template복사
                        // 코드의 zip으로 다운 받는 대신, 태그나 릴리즈로 받는게 안전함
                        sh """
                            rm -rf ./manifest_repo && rm -rf ./template && rm -rf ./template.zip
                            echo git clone repo && git clone https://${env.GITHUB_USERNAME}:${TOKEN}@github.com/${env.GITHUB_USERNAME}/${params.application_name}-manifest.git ./manifest_repo
                            echo download template && curl -o template.zip -L  https://github.com/${env.GITHUB_USERNAME}/argocd-manifest-template/archive/refs/heads/main.zip
                            echo uncompress template &&  unzip -j template.zip -d ./template
                            cp ./template/* ./manifest_repo
                        """

                        // git push
                        sh """
                            cd ./manifest_repo
                            git status

                            git config --local user.email "jenkins_bot@demo.com"
                            git config --local user.name "jenkins_bot"

                            git add -A
                            git commit --allow-empty -m "copy template"

                            git push
                        """

                        // 작업파일 삭제
                        sh """
                            rm -rf ./manifest_repo && rm -rf ./template && rm -rf ./template.zip
                        """
                        }
                    } else {
                        println "argo_manifest_repo(${argo_manifest_repo}) alreay exist. skip create manifest repo."
                    }
                }
            }
        }
        stage("update manifest") {
            steps {
                withCredentials([string(credentialsId: 'github_token', variable: 'TOKEN')]) {
                    sh "rm -rf ./manifest_repo"
                    sh "echo git clone repo && git clone https://${env.GITHUB_USERNAME}:${TOKEN}@github.com/${env.GITHUB_USERNAME}/${params.application_name}-manifest.git ./manifest_repo"

                    // update helm chart values.yaml
                    sh """ 
                        cd ./manifest_repo

                        export docker_image=${env.DOCKER_USERNAME}/demo-image:${BUILD_NUMBER}
                        yq -i '.image = strenv(docker_image)' values.yaml
                    """

                    // git push
                    sh """
                        cd ./manifest_repo
                        git status

                        git config --local user.email "jenkins_bot@demo.com"
                        git config --local user.name "jenkins_bot"

                        git add -A
                        git commit --allow-empty -m "update manifest"

                        git push
                    """
                }
            }
        }
        stage("Sync with argocd") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'argocd-cred', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh """
                        echo "y" | argocd login ${env.ARGOCD_URL} --username ${username} --password ${password} --insecure

                        # todo. app create를 계속해도 이상없는지
                        argocd app create ${params.application_name} \
                        --repo https://github.com/${env.GITHUB_USERNAME}/${params.application_name}-manifest.git \
                        --path . \
                        --dest-namespace default \
                        --dest-server https://kubernetes.default.svc

                        echo "argocd app sync ${params.application_name} -o --prune"
                        argocd app sync ${params.application_name}
                    """
                }
            }
        }
    }
}