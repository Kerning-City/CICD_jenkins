# 🚀 Jenkins를 사용한 SpringBoot 자동 배포

## 👨‍👨‍👦‍👦 Team

👥 **팀명** : 커닝시티

|<img src="https://avatars.githubusercontent.com/u/71498489?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/123963462?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/95984922?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/143996082?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|HanJH<br/>[@letsgojh0810](https://github.com/letsgojh0810)|태규<br/>[@EOTAEGYU](https://github.com/EOTAEGYU)|nahong_c<br/>[@HongChan1412](https://github.com/HongChan1412)|Jihoon Kim<br/>[@wild-turkey](https://github.com/wild-turkey)|

---

## 📌 프로젝트 목표
- **Jenkins Pipeline**과 **SSH Key 기반 자동화**를 활용해  로컬 PC(Jenkins 컨테이너)에서 빌드된 JAR 파일을  **내 PC 내부의 myserver02 (Ubuntu 서버)** 또는  **같은 네트워크에 있는 다른 사용자의 Ubuntu 서버**  로 자동 전송하고, 해당 서버에서 파일 변경을 감지하여 자동으로 애플리케이션을 재실행하도록 구현하는 것

---

## ✅ 시스템 아키텍처

```
[GitHub Repository]
       |
       ▼ (웹훅 또는 트리거)
[Jenkins (Docker)]
       |
       |   - Git Clone
       |   - Gradle Build
       ▼
[빌드된 JAR 파일]
       |
       ▼ (SSH Key 기반 SCP)
[Target Server (myserver02 또는 다른 사용자 Ubuntu)]
       |
       ▼ (inotify 감지 스크립트)
[자동 실행 및 배포 완료]
```

---

## ✅ 주요 구성

| 구성 요소            | 설명                                                            |
| ---------------- | ------------------------------------------------------------- |
| Jenkins Pipeline | GitHub `main` 브랜치 감지 후 자동 빌드 및 JAR 파일 전송                      |
| SSH Key 인증       | `sshpass` 사용 없이 Jenkins 컨테이너 내에서 생성한 SSH Key 를 서버에 등록하여 자동 접속 |
| 파일 전달 대상         | 내 PC의 myserver02 또는 같은 네트워크 내 다른 사용자의 Ubuntu 서버               |
| 자동 실행 감지         | 대상 서버에서 `inotify-tools`를 통해 JAR 파일 변경 시 자동 실행 감지 및 재시작 처리     |
| 실행 포트            | `8081` 포트에서 자동 실행 설정 (필요시 변경 가능)                              |

---

## ✅ Jenkins Pipeline 구조

1. **Clone Repository**
   - GitHub에서 최신 코드를 가져옵니다.

2. **Build JAR**
   - Gradle을 이용해 `step07_CICD` 프로젝트를 빌드합니다.

3. **Transfer JAR to Target Server**
   - SSH Key 기반 `scp`를 통해 대상 서버로 JAR 파일을 전송합니다.

> 📂 Jenkins Pipeline 코드 예시는 아래 참고

---

## ✅ Jenkins Pipeline 코드 (예시)
```groovy
pipeline {
    agent any
    
    environment {
        DEST_IP = "192.168.0.74"     // 전송할 대상 서버 IP
        DEST_PATH = "/home/ubuntu/appjar/"
        JAR_NAME = "app.jar"
        SSH_PORT = "22"
        SSH_KEY = "/var/jenkins_home/.ssh/id_rsa"
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/EOTAEGYU/20250324a.git'
            }
        }
        
        stage('Build JAR') {
            steps {
                dir('./step07_CICD') {                   
                   sh 'chmod +x gradlew'    
	               sh './gradlew clean build'
                   echo "JAR build completed"
                }
            }
        }  
        
        stage('Transfer JAR to Target Server') {
            steps {
                script {
                    def jarFile = 'step07_CICD/build/libs/step07_CICD-0.0.1-SNAPSHOT.jar'
                    
                    sh """
                        ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no -p ${SSH_PORT} ubuntu@${DEST_IP} "mkdir -p ${DEST_PATH}"
                        scp -i ${SSH_KEY} -P ${SSH_PORT} ${jarFile} ubuntu@${DEST_IP}:${DEST_PATH}${JAR_NAME}
                    """
                }
            }
        }
    }
}
```

---

## ✅ 대상 서버 inotify 감지 스크립트 (watch_jar.sh)
```bash
#!/bin/bash

JAR_PATH="/home/ubuntu/appjar/app.jar"
LOG_PATH="/home/ubuntu/appjar/app.log"
PORT=8081

echo "Watching for changes to $JAR_PATH..."

while true
do
    inotifywait -e close_write $JAR_PATH

    echo "Change detected in $JAR_PATH. Restarting the application..."

    pkill -f "${JAR_PATH}" || true
    sleep 2

    nohup java -jar $JAR_PATH --server.port=$PORT > $LOG_PATH 2>&1 &
    echo "Application restarted on port $PORT."
done
```
> ✅ 실행 방법
```bash
chmod +x watch_jar.sh
nohup ./watch_jar.sh > watch_jar_monitor.log 2>&1 &
```

---

## ✅ 추가로 적용한 사항
- Jenkins 컨테이너 내부에서 SSH Key 발급 및 각 서버 등록
- SSH 포트는 `22`번 고정
- myserver02 및 사용자 서버에서 `inotify-tools` 설치 완료
- 대상 서버마다 `authorized_keys`에 Jenkins 컨테이너 공개 키 등록 완료
- 다른 사용자의 서버에도 동일 방식으로 전달 및 자동 실행 가능하도록 구현

---

## ✅ 프로젝트 구성 완료 후 테스트 결과
| 항목                     | 상태                          |
|------------------------|-----------------------------|
| Jenkins Pipeline 빌드   | ✅ 성공                           |
| myserver02 파일 전달    | ✅ SCP 전송 및 자동 실행 정상 동작      |
| 다른 사용자 서버 전달    | ✅ 같은 네트워크 사용자 서버에 SCP 및 감지 실행 완료 |
| 자동 실행 스크립트 감지 | ✅ jar 변경 시 자동으로 실행 재시작 확인 완료 |

