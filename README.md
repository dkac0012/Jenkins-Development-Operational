# Jenkins-Development-Operational

## 공부 목적
Jenkins를 통해 개발 서버와 운영 서버를 나누어 code pipeline을 자동화 하는 경험을 얻고자 하였습니다.

## Jenkins 
Jenkins는 CI/CD를 위한 오픈 소스 도구로, 소프트웨어 개발 생명주기를 자동화한다.
 
Jenkins를 사용하면 코드 변경 사항이 발생할 때마다 자동으로 빌드, 테스트, 배포 과정을 수행할 수 있기 때문에 개발자는 코드 변경에 따른 빌드 및 테스트 과정을 수동으로 수행할 필요가 없어지며, 신속하게 개발에 대한 코드 검증 및 배포할 수 있고, 이 과정에서 휴먼 에러를 방지할 수 있다.


### Jenkins 사용목적


## 구현

### github 연동
#### ngrok

```window
Forwarding  https://39fd-118-131-63-236.ngrok-free.app -> http://127.0.0.1:8888
```

#### github-webhook

![image](https://github.com/user-attachments/assets/6304a798-8eb7-488d-916f-f3f09d3bbfb6)

```java
@Slf4j
@RestController
public class ProcessController {
	
	@GetMapping("/test")
	public String reqRes() {
		log.info("요청 수락 ~~~");
		return "linux 서버에서 실행되는 app";
	}
}
```
코드 수정

### 개발 서버

**monitoring.sh**
```bash
#!/bin/bash

# JAR 파일 경로 설정
JAR_FILE="./SpringApp-0.0.1-SNAPSHOT.jar"

SH_FILE="./movescp.sh"

# COOLDOWN 중복 실행 방지 대기 시간 (예: 10초)
COOLDOWN=10
LAST_RUN=0

# 파일 수정 감지 및 .sh 파일 실행
inotifywait -m -e close_write "$JAR_FILE" |
while read -r directory events filename; do
    CURRENT_TIME=$(date +%s)

    # 마지막 실행 후 지정된 시간이 지났는지 확인
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        echo "$(date): $filename 파일이 수정되었습니다."  
        # .sh 파일 실행
        bash "$SH_FILE"
        # 마지막 실행 시간 업데이트
        LAST_RUN=$CURRENT_TIME
    else
        echo "$(date): 쿨다운 기간 중입니다. "
    fi
done
```

**movescp.sh**
```bash
#!/bin/bash

filename="SpringApp-0.0.1-SNAPSHOT.jar"
# 변수 설정
LOCAL_FILE_PATH="$(pwd)/$filename"  # 로컬 파일 경로
REMOTE_USER="username"                    # 원격 서버 사용자명
REMOTE_HOST="10.0.2.21"           # 원격 서버 IP 또는 도메인
REMOTE_DIR="/home/username/runServer"    # 원격 서버 대상 경로

# SCP 명령 실행
scp $LOCAL_FILE_PATH $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR

# 전송 결과 출력
if [ $? -eq 0 ]; then
    echo "파일 전송 성공!"
else
    echo "파일 전송 실패!"
fi
```

**jar 파일 시간 확인**
```bash
username@servername:~/appjardir$ ll
total 19584
drwxr-xrwx  2 root     root         4096 Oct  1 14:09 ./
drwxr-x--- 32 username username     4096 Oct  2 10:04 ../
-rw-rw-r--  1 username username     1980 Oct  1 10:42 app.log
-rw-rw-r--  1 username username      751 Oct  1 11:57 monitoring.sh
-rw-rw-r--  1 username username      540 Oct  1 12:16 movescp.sh
-rwxr-xr-x  1 username username 20031439 Oct  1 17:17 SpringApp-0.0.1-SNAPSHOT.jar*
```

**PEM&SCP**
SCP를 사용할때 해당 접속 host의 passwd를 요구한다 이때 passwd없이 접속하기위해 PEM키가 사용된다.
```bash
key 생성
ssh-keygen -t rsa -b 4096

각 파일 생성 확인
cat ~/.ssh/*

ssh-copy-id username@'targetip'
```


**monitoring.sh 실행 결과**
```bash
username@servername:~/appjardir$ bash monitoring.sh 
Setting up watches.
Watches established.
Wed Oct  2 12:31:21 PM KST 2024:  파일이 수정되었습니다.
SpringApp-0.0.1-SNAP 100%   19MB  40.7MB/s   00:00    
파일 전송 성공!
Wed Oct  2 12:31:24 PM KST 2024: 쿨다운 기간 중입니다.
```


### 운영 서버

**change.sh**
```bash
#!/bin/bash

# JAR 파일 경로 설정
JAR_FILE="./SpringApp-0.0.1-SNAPSHOT.jar"

# 실행할 .sh 파일 경로 설정
SH_FILE="./autorunning.sh"

# COOLDOWN 중복 실행 방지 대기 시간 (예: 10초)
COOLDOWN=10
LAST_RUN=0

# 파일 수정 감지 및 .sh 파일 실행
inotifywait -m -e close_write "$JAR_FILE" |
while read -r directory events filename; do
    CURRENT_TIME=$(date +%s)

    # 마지막 실행 후 지정된 시간이 지났는지 확인
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        echo "$(date): $filename 파일이 수정되었습니다."  # 수정 시간 로그 추가
        # .sh 파일 실행
        bash "$SH_FILE"
        # 마지막 실행 시간 업데이트
        LAST_RUN=$CURRENT_TIME
    else
        echo "$(date): 쿨다운 기간 중입니다. 실행하지 않음."
    fi
done
```

**autorunning.sh**
```bash
#!/bin/bash

# Spring Boot 애플리케이션 재시작
# 기존 8080 포트 사용 중인 프로세스 종료
if  lsof -i :8899 > /dev/null; then
  # 8999 포트가 사용 중일 경우 이전 프로세스를 종료
  kill -9 $(lsof -t -i :8899)
  echo '정상적으로 종료되었습니다.'
fi

# 백그라운드에서 새로 실행
# > $DEPLOY_DIR/app.log : 애플리케이션 로그를 app.log 파일에 저장하도록 구성
nohup java -jar SpringApp-0.0.1-SNAPSHOT.jar > app.log 2>&1 &

echo "배포완료 및 재실행됩니다."
```

**jar 파일 시간 확인**
```bash
username@servername:~/runServer$ ll
total 19584
drwxrwxr-x 2 username username     4096 Oct  1 12:12 ./
drwxr-x--- 7 username username     4096 Oct  1 11:53 ../
-rw-rw-r-- 1 username username     2728 Oct  1 16:34 app.log
-rw-rw-r-- 1 username username      540 Oct  1 12:26 autorunning.sh
-rw-rw-r-- 1 username username      841 Oct  1 12:00 change.sh
-rwxr-xr-x 1 username username 20031434 Oct  1 16:28 SpringApp-0.0.1-SNAPSHOT.jar*
```

**change.sh 실행 결과**
```bash
username@servername:~/runServer$ bash change.sh
Setting up watches.
Watches established.
Wed Oct  2 12:31:22 PM KST 2024:  파일이 수정되었습니다.
정상적으로 종료되었습니다.
배포완료 및 재실행됩니다.
```

**서비스 확인**
```bash
username@servername:~/runServer$ lsof -i :8899
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    22175 username    9u  IPv6  52494      0t0  TCP *:8899 (LISTEN)
```

![image](https://github.com/user-attachments/assets/21e81769-a2f2-4535-8048-76e8ae099f9e)
![image](https://github.com/user-attachments/assets/a2cbe126-88c9-469d-b0c3-411d25d24c36)

