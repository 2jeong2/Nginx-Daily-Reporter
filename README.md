# 📌 Nginx Log Monitoring Project

## 📖 프로젝트 소개
Nginx 웹 서버의 **접속 로그와 에러 로그를 모니터링**하여 서비스 가용성 및 보안 이벤트를 빠르게 파악하기 위한 프로젝트 

특히 **SSH 로그인 실패 탐지, 접속 현황 분석, 리소스 사용량 확인** 등을 자동화 스크립트로 구현함

---

## 👨‍👩‍👧‍👦 팀원 소개

| 이름 | 사진 | GitHub |
|------|------|---------|
| 이정이 | ![사진](./images/hong.jpg) | [github.com/hong](https://github.com/hong) |
| 신기범 | ![사진](./images/kim.jpg) | [github.com/shin-kibeom](https://github.com/shin-kibeom) |


---

## 📅 진행 순서

### 1️⃣  - Nginx 설치 및 기본 로그 확인  
### 2️⃣  - SSH 로그인 실패 탐지  
### 3️⃣  - 로그 분석 자동화 스크립트  
### 4️⃣  - 실시간 모니터링  

 
## ⚙️ 1. Nginx 설치 및 기본 로그 확인

```bash
  sudo apt update
  sudo apt install nginx      #nginx 설치
  sudo systemctl start nginx  
  sudo systemctl status nginx 
```
```bash
  cat /var/log/nginx/access.log
  cat /var/log/nginx/error.log
```

## 🖥️ 2.  SSH 로그인 실패 탐지
  grep "Failed password" /var/log/auth.log


## 🖥️ 3. 로그 분석 스크립트 작성

```bash
sudo nano /usr/local/bin/nginx_log_analysis.sh   # 스크립트 파일 생성
sudo chmod +x /usr/local/bin/nginx_log_analysis.sh  # 실행 권한 부여 (실행 권한이 없을시)
```

### 📄 `nginx_log_analysis.sh`

```bash
#!/bin/bash

LOGFILE="/var/log/nginx/access.log"           # nginx access log 파일 가져오기
OUTPUT="/var/log/nginx/analysis_append.log"   # /var/log/nginx 경로의 analysis_append.log에 기록

# 분석 시작
echo "===== $(date '+%Y-%m-%d %H:%M:%S') 분석 시작 =====" >> $OUTPUT

# IP별 요청 수
echo "==== IP별 요청 수 ====" >> $OUTPUT
awk '{print $1; fflush()}' "$LOGFILE" | sort | uniq -c | sort -nr >> $OUTPUT

# 상태 코드별 요청 수
echo "==== 상태 코드별 요청 수 ====" >> $OUTPUT
awk '{print $9; fflush()}' "$LOGFILE" | sort | uniq -c | sort -nr >> $OUTPUT

# 요청 URL 상위 10
echo "==== 요청 URL 상위 10 ====" >> $OUTPUT
awk '{print $7; fflush()}' "$LOGFILE" | sort | uniq -c | sort -nr | head -10 >> $OUTPUT

# 분석 끝
echo "===== 분석 끝 =====" >> $OUTPUT
echo "" >> $OUTPUT
```

---

### Crontab 설정

```bash
sudo crontab -e    # 2번(vim) 편집기 선택
```

## 설정 예시
```bash
0 0 * * * /usr/local/bin/nginx_log_analysis.sh # 매일 자정마다 스크립트 실행
```

```bash
* * * * * /usr/local/bin/nginx_log_analysis.sh # 1분마다 실행 (테스트용)
```

---

## ▶️ 4. 실행 및 결과 확인

수동 실행:

```bash
sudo /usr/local/bin/nginx_log_analysis.sh
```

로그 확인:

```bash
tail -F /var/log/nginx/analysis_append.log
```
<img width="520" height="600" alt="image" src="https://github.com/user-attachments/assets/9a04a1b1-d25a-451e-b557-33771df8aef8" />

## 🌐 5. VirtualBox 네트워크 설정 (참고)

- VM 네트워크를 **Bridged Adapter** 로 변경
<img width="520" height="520" alt="image" src="https://github.com/user-attachments/assets/24a9097f-2756-4d39-b31d-9da728dbb2a5" />

- 실제 pc와 연결 된 LAN에 직접 연결되어 dhcp로 IP 자동 할당
- 같은 네트워크의 다른 PC(예: 노트북)에서 접속 가능

---

## 📷 결과 화면



---

## 🛠️ 트러블슈팅 & 💡 회고

### 👤 이정이
문제   : SSH 로그인 실패 로그가 스크립트에서 출력되지 않음  
원인   : grep 패턴에 불필요한 [[:space:]] 추가 → 매칭 실패  
해결   : 단순 문자열 "Failed password"로 수정  
회고   : 로그 포맷이 일정하지 않아 파싱 로직을 다듬는 데 시간이 걸렸습니다. awk와 grep 활용 능력이 많이 늘었습니다.  

---

### 👤 신기범
문제   : 스크립트 실행 시 출력이 끊여 가독성이 떨어짐  
원인   : tail 명령어는 기본적으로 파일의 마지막 10줄을 출력  
해결   : tail -n 0 -f 옵션으로 새로 append되는 log만 출력
회고   : 로그 파일을 시각화 하는 도구 사용을 못 해서 아쉽고 로그 정보에 비해 유의미한 데이터 분석이 부족

---

## 📊 로그 흐름 구조 (Mermaid Diagram)

```mermaid
flowchart TD
    A[사용자 요청] --> B[Nginx Access Log]
    A --> C[Nginx Error Log]
    D[SSH 로그인 시도] --> E[auth.log]

    B --> F[log_monitor.sh]
    C --> F[log_monitor.sh]
    E --> F[log_monitor.sh]

    F --> G[로그 분석 결과 출력]
    G --> H[관리자 알림 / 보안 강화]
