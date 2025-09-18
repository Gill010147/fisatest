## 🚀 Spring Boot CI/CD Pipeline with Jenkins & Docker
Jenkins를 활용하여 CI/CD 파이프라인을 직접 구성하고 검증한 실습입니다.
- GitHub 저장소 변경 사항을 자동으로 감지  
- Gradle 빌드 → JAR 생성 → 아카이빙 자동화  
- Jenkins의 핵심 기능(CI/CD) 실습 및 검증

## 🧰 기술 스택

| Ubuntu | Jenkins | Java 17 | Gradle | GitHub | Ngrok |
| --- | --- | --- | --- | --- | --- |
| <img src="https://cdn.simpleicons.org/ubuntu" width="36"> | <img src="https://cdn.simpleicons.org/jenkins" width="36"> | <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/java/java-original.svg" width="36"> | <img src="https://cdn.simpleicons.org/gradle/02303A" width="36"> | <img src="https://avatars.githubusercontent.com/u/22289824?s=200&v=4" width="36"> | <img src="https://raw.githubusercontent.com/gilbarbara/logos/main/logos/ngrok.svg" width="60"> |


## 📚 CI/CD 파이프라인 구축 가이드
### 1. Jenkins 컨테이너 설치 (바인드 마운트)
호스트 머신에 Jenkins 데이터를 저장하기 위해 호스트 디렉터리를 생성하고, 이를 Jenkins 컨테이너의 `/var/jenkins_home`에 바인드 마운트함.


- 호스트에 Jenkins 홈 디렉터리 생성 및 권한 설정
- Jenkins 컨테이너의 기본 사용자 UID/GID는 1000
```
sudo mkdir -p /srv/jenkins
sudo chown -R 1000:1000 /srv/jenkins
```

- Jenkins 컨테이너 실행 (호스트 포트 8080 연결)
```
docker run -d \
  --name myjenkins \
  -p 8080:8080 \
  -v /srv/jenkins:/var/jenkins_home \
  jenkins/jenkins:lts-jdk17
```

### 2. Jenkins 초기 설정
- 브라우저에서 http://<호스트 IP>:8080에 접속하여 초기 설정을 진행.

- 호스트에서 초기 비밀번호 확인
```
cat /srv/jenkins/secrets/initialAdminPassword
```
- 또는 컨테이너 내부에서 확인
```
docker exec myjenkins cat /var/jenkins_home/secrets/initialAdminPassword
비밀번호 입력 후, Install suggested plugins를 선택하고 관리자 계정을 생성.
```

### 3. Jenkins 파이프라인 잡(Job) 생성
> Jenkins 대시보드에서 New Item을 클릭.

> 아이템 이름(예: step03_teamArt)을 입력하고 Pipeline을 선택.

> Pipeline 탭으로 이동하여 Definition 항목을 Pipeline script로 변경.

> Build Triggers 섹션에서 GitHub hook trigger for GITScm polling 옵션을 체크하고 저장.

## 🦴Jenkinsfile (Pipeline Script)
- 이 파이프라인은 Checkout, Build, Artifact 저장 단계를 수행하며 Gradle과 Maven 프로젝트를 자동으로 감지.

```
// Jenkinsfile
pipeline {
    agent any // 어떤 Jenkins 에이전트에서든 이 파이프라인을 실행합니다.

    // GitHub Push가 발생하면 자동으로 빌드를 시작하는 Webhook 트리거입니다.
    triggers { githubPush() }

    // 파이프라인 전체에서 사용할 환경 변수를 정의합니다.
    environment {
        // 1. GitHub 설정
        GITHUB_REPO   = 'https://github.com/kohtaewoo/fisatest.git' // 본인의 GitHub 저장소 URL로 변경하세요.
        BRANCH_NAME   = 'main'                                       // 사용할 브랜치 이름입니다.
        PROJECT_PATH  = 'step03_JPAGradle'                           // Git 저장소 내의 실제 프로젝트 디렉토리 이름입니다.

        // 2. 배포 대상 서버(myserver02) 정보
        // myserver02의 IP 주소 또는 호스트 이름을 정확하게 입력하세요.
        // Jenkins 컨테이너에서 접근 가능한 IP여야 합니다.
        DEPLOY_HOST   = 'ubuntu@10.0.2.20'
        
        // myserver02에 애플리케이션을 배포할 경로와 systemd 서비스 이름입니다.
        APP_DIR       = '/opt/step03/app'
        SERVICE       = 'step03' 
        
        // 3. Jenkins SSH Credentials ID
        // Jenkins에 등록한 'SSH Username with private key' Credential의 ID입니다.
        SSH_CREDENTIAL_ID = 'myserver02-ssh-key' 
    }

    stages {
        // Stage 1: 소스 코드 가져오기
        stage('Checkout') {
            steps {
                git branch: "${BRANCH_NAME}", url: "${GITHUB_REPO}"
                echo "Source code checkout successful from ${BRANCH_NAME} branch."
                sh "ls -al ${PROJECT_PATH}"
            }
        }

        // Stage 2: 프로젝트 빌드 (Gradle 또는 Maven)
        stage('Build') {
            steps {
                script {
                    if (fileExists("${PROJECT_PATH}/gradlew")) {
                        echo "Gradle project detected. Starting build..."
                        dir("${PROJECT_PATH}") {
                            sh 'chmod +x gradlew'
                            sh './gradlew clean build -x test'
                        }
                    } else if (fileExists("${PROJECT_PATH}/pom.xml")) {
                        echo "Maven project detected. Starting build..."
                        dir("${PROJECT_PATH}") {
                            sh 'mvn -B -DskipTests clean package'
                        }
                    } else {
                        error "Build failed: Neither gradlew nor pom.xml found."
                    }
                    echo "Build successful."
                }
            }
        }

        // Stage 3: 빌드 결과물(JAR 파일) 확인
        stage('Verify Artifact') {
            steps {
                echo "Verifying created artifacts..."
                sh "ls -al ${PROJECT_PATH}/build/libs/ || true"
                sh "ls -al ${PROJECT_PATH}/target/ || true"
            }
        }

        // Stage 4: myserver02로 배포 및 애플리케이션 재시작
        stage('Deploy') {
            steps {
                sshagent(credentials: [SSH_CREDENTIAL_ID]) {
                    script {
                        // ==========================================================
                        // === 수정된 부분: '-plain.jar' 파일을 제외하고 하나의 JAR만 선택 ===
                        // ==========================================================
                        def jarFile = sh(script: "find ${PROJECT_PATH}/build/libs/ -name '*.jar' | grep -v -- '-plain.jar' || find ${PROJECT_PATH}/target/ -name '*.jar'", returnStdout: true).trim()

                        if (jarFile.isEmpty()) {
                            error "Deploy failed: JAR file not found."
                        }

                        echo "Deploying ${jarFile} to ${DEPLOY_HOST}..."

                        sh """
                            # 1. myserver02에 배포 디렉토리가 없으면 생성합니다.
                            ssh -o StrictHostKeyChecking=no ${DEPLOY_HOST} 'sudo mkdir -p ${APP_DIR}'

                            # 2. 빌드된 JAR 파일을 myserver02의 임시 경로로 복사합니다.
                            scp -o StrictHostKeyChecking=no "${jarFile}" ${DEPLOY_HOST}:/tmp/app.jar

                            # 3. 임시 경로의 JAR 파일을 최종 배포 경로로 옮기고 권한을 설정합니다.
                            ssh -o StrictHostKeyChecking=no ${DEPLOY_HOST} 'sudo mv /tmp/app.jar ${APP_DIR}/app.jar && sudo chown ubuntu:ubuntu ${APP_DIR}/app.jar'

                            # 4. myserver02에 등록된 systemd 서비스를 재시작하고 상태를 확인합니다.
                            ssh -o StrictHostKeyChecking=no ${DEPLOY_HOST} 'sudo systemctl restart ${SERVICE} && sleep 3 && sudo systemctl status ${SERVICE} --no-pager'
                        """
                    }
                }
            }
        }
    }

    // 파이프라인 실행 후 처리
    post {
        success {
            echo "Pipeline successful! Deployment to ${DEPLOY_HOST} is complete."
        }
        failure {
            echo "Pipeline failed. Please check the console output for errors."
        }
    }
}

```

### 4. GitHub Webhook 연동
- 빌드할 GitHub 저장소의 Settings > Webhooks > Add webhook으로 이동.

- Payload URL에 http://<Jenkins 서버 IP>:8080/github-webhook/을 입력.

- Content type을 application/json으로 설정.

- Which events would you like to trigger this webhook? 에서 Just the push event를 선택하고 저장.


### 5. Git Push로 파이프라인 트리거하기
- 로컬에서 코드를 수정한 후 git push를 실행하면 Webhook이 Jenkins를 호출하여 파이프라인이 자동으로 시작.


- Git 사용자 정보 설정 (최초 1회)
```
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

- 변경사항 추가, 커밋, 푸시
```
git add .
git commit -m "feat: Update application logic and trigger CI pipeline"
git push origin main
```

### 6. 호스트에서 빌드 산출물 확인
파이프라인이 성공적으로 완료되면, 바인드 마운트된 호스트 경로에서 .jar 파일을 확인 가능.


- 워크스페이스 루트에 복사된 JAR 파일 확인
```
ls -al /srv/jenkins/workspace/step03_teamArt/*.jar
```
<img width="1326" height="66" alt="image" src="https://github.com/user-attachments/assets/aea895cc-3eae-4933-8d3c-202e4434c3a7" />


- 또는 원본 빌드 경로 확인 (Gradle 기준)
```
ls -al /srv/jenkins/workspace/step03_teamArt/step03_JPAGradle/build/libs/
```
<img width="980" height="127" alt="image" src="https://github.com/user-attachments/assets/24f5ffc7-c83b-4fc0-a8d7-5123707d51c0" />

### 7. 애플리케이션 컨테이너 실행
- 호스트에 저장된 .jar 파일을 OpenJDK 컨테이너에 마운트하여 애플리케이션을 실행.
- 환경 변수로 산출물 경로 지정
```
export APP_JAR_PATH=/srv/jenkins/workspace/step03_teamArt
```

- Docker 컨테이너 실행 (호스트 포트 8900 연결)
>  --mount 옵션은 호스트의 JAR 파일을 컨테이너의 /app 디렉터리에 읽기 전용으로 마운트.
```
docker run -d \
  --name step03-app \
  -p 8900:8900 \
  --mount type=bind,source=${APP_JAR_PATH},target=/app,readonly \
  openjdk:17-jdk-slim \
  java -jar /app/*.jar
```

- 애플리케이션 로그 확인
'''
docker logs -f step03-app
'''
- application.properties 파일에 server.port = 8900으로 설정되어 있으므로 호스트의 8900 포트를 컨테이너의 8900 포트와 연결.

# 8. 실행 결과 (Pipeline Result)

### 8.1 Stage View
Jenkins 파이프라인이 **Checkout → Build JAR →  Set Permission → Archive** 단계까지 정상적으로 동작했으며,  
빌드 결과물 JAR 파일이 성공적으로 아카이빙됨.

<img src="https://i.postimg.cc/2SPJFF1G/image.png" alt="Jenkins Pipeline Stage View" width="700">

<br>

### 8.2 산출물 경로 (Artifact)
빌드된 JAR 파일은 Jenkins 워크스페이스에 생성:

```
~/jenkis-test/workspace/step03_teamArt/build/libs/step05_GradleBuild-0.0.1-SNAPSHOT.jar
```
<br>

### 8.3 실행 (Run on Ubuntu)
```bash
cd ~/jenkis-test/workspace/step03_teamArt/build/libs
```

### 8.4 실행 화면 (Running App)

> Jenkins 빌드 산출물인 Jar 실행

<img src="https://i.postimg.cc/50qtLSmQ/image.png" alt="App running on 8081" width="700">

<br>

> 웹에서 localhost 접속

<img src="https://i.postimg.cc/1RFjsY3K/image.png" alt="App running on 8081" width="700">

<br>


## 📝 트러블슈팅

**1) Webhook**

- GitHub Webhook → **Recent Deliveries**에서 Response **200** 확인
- **Payload URL**이 `/github-webhook/` 로 끝나는지 확인
- Jenkins 플러그인/설정 확인: **Git**, **GitHub**, (필요시) Manage Jenkins → Configure System → GitHub 서버 추가
- 방화벽/보안그룹에서 **8080** 오픈
- 로컬이면 **ngrok https 8080** 으로 공개 URL 사용

**2) 권한/퍼미션 문제**

- 호스트 바인드 마운트 디렉터리는 Jenkins UID/GID(1000)에 소유권 부여
    
    ```bash
    sudo chown -R 1000:1000 /srv/jenkins
    
    ```
    
- 호스트에서 Docker 명령 권한이 없으면:
    
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    
    ```

## 📂 디렉터리 예시

```
fisatest/
└─ step03_JPAGradle/
   ├─ build/
   │  └─ libs/
   │     └─ app-0.0.1-SNAPSHOT.jar
   ├─ src/...
   ├─ gradlew
   └─ settings.gradle

```
