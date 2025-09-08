# 📌 Nginx Log Monitoring  

## 📖 Nginx Log Monitoring 소개  
Nginx 웹 서버의 **접속 로그와 에러 로그를 모니터링**하여 서비스 가용성 및 보안 이벤트를 빠르게 파악하기 위한 프로젝트 

특히 **SSH 로그인 실패 탐지, 접속 현황 분석** 등을 자동화 스크립트로 구현함  

---  
  
## 👨‍👩‍👧‍👦 팀원 소개

| 이정이 | 신기범 |
|--------|--------|
| <img src="https://github.com/2jeong2.png" width="100"/> | <img src="https://github.com/shin-kibeom.png" width="100"/> |
| [이정이](https://github.com/2jeong2) | [신기범](https://github.com/shin-kibeom) |
  
---  
  
## 📅 진행 순서

### 1️⃣  - Nginx 설치 및 기본 로그 확인  
### 2️⃣  - SSH 로그인 실패 탐지  
### 3️⃣  - 로그 분석 자동화 스크립트  
### 4️⃣  - 실행 및 결과 확인    

 
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
Ubuntu 24.04는 /var/log/auth.log 대신 journald 사용. ssh/sshd 둘 다 조회   
journalctl -u ssh -u sshd -S today -U tomorrow --no-pager | grep -F "Failed password"

### 🔎 SSH 로그인 실패 로그 수집 방법

- `journalctl -u ssh -u sshd -S today -U tomorrow --no-pager`  
  → 오늘 하루 SSH 관련 로그만 출력  

- `grep -F "Failed password"`  
  → 실패한 로그인 시도만 추출  

- `awk '{print $11}'`  
  → 실패한 **사용자 계정** 확인  

- `awk '{print $13}'`  
  → 실패한 **IP 주소** 확인  

- `sort | uniq -c | sort -nr`  
  → 개수 세고 내림차순 정렬  

- `|| echo "(오늘자 실패 없음)"`  
  → 실패 로그 없을 때 메시지 출력


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

# 오늘 날짜 기준 SSH 로그인 실패 로그 제목 출력
  echo "==== SSH 로그인 실패 (오늘: $(date '+%Y-%m-%d')) ====" >> "$OUTPUT"

# 오늘 하루 SSH 실패 로그 전체 출력 (없으면 "오늘자 실패 없음" 출력)
journalctl -u ssh -u sshd -S today -U tomorrow --no-pager \
| grep -F "Failed password" >> "$OUTPUT" || echo "(오늘자 실패 없음)" >> "$OUTPUT"


# 사용자별 SSH 로그인 실패 횟수 집계
echo "==== SSH 실패 - 사용자별 (오늘) ====" >> "$OUTPUT"
journalctl -u ssh -u sshd -S today -U tomorrow --no-pager \
| grep -F "Failed password" \
| awk '{print $11}' | sort | uniq -c | LC_ALL=C sort -nr >> "$OUTPUT" || true


# IP별 SSH 로그인 실패 횟수 집계
echo "==== SSH 실패 - IP별 (오늘) ====" >> "$OUTPUT"
journalctl -u ssh -u sshd -S today -U tomorrow --no-pager \
| grep -F "Failed password" \
| awk '{print $13}' | sort | uniq -c | LC_ALL=C sort -nr >> "$OUTPUT" || true

# 분석 끝
echo "===== 분석 끝 =====" >> $OUTPUT
echo "" >> $OUTPUT
```


## ➕ fake traffic  
### 1. 스크립트 작성
```bash
  #!/bin/bash
  URL="http://localhost"

  while true; do
    # 정상 요청
    curl -s -o /dev/null $URL/
    # 존재하지 않는 페이지(404)
    curl -s -o /dev/null $URL/not_found_$RANDOM.html
    # 보호된 페이지(401/403)
    curl -s -o /dev/null $URL/protected/
    sleep 5   # 5초 간격으로 반복
  done

```
### 2. 실행 권한 부여
```bash
chmod +x ~/fake_traffic.sh
```

### 3. 백그라운드 실행 (로그를 계속 쌓게 함)
```bash
nohup ~/fake_traffic.sh >/dev/null 2>&1 &
```
### 4. 동작 확인
```bash
ps aux | grep fake_traffic
```

### 5. 중지 방법 (프로세스 종료)
```bash
pkill -f fake_traffic.sh
```
### 또는
```bash
kill -9 <PID>
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

## 📷 4. 실행 및 결과 확인

수동 실행:

```bash
sudo /usr/local/bin/nginx_log_analysis.sh
```

로그 확인:

```bash
tail -F /var/log/nginx/analysis_append.log
```
<img width="520" height="600" alt="image" src="https://github.com/user-attachments/assets/9a04a1b1-d25a-451e-b557-33771df8aef8" />
<img width="936" height="612" alt="ssh_failed" src="https://github.com/user-attachments/assets/ffdfe79a-c009-463a-9fb4-aaaaf2719222" />


## 🌐 5. VirtualBox 네트워크 설정 (참고)

- VM 네트워크를 **Bridged Adapter** 로 변경
<img width="520" height="520" alt="image" src="https://github.com/user-attachments/assets/24a9097f-2756-4d39-b31d-9da728dbb2a5" />

- 실제 pc와 연결 된 LAN에 직접 연결되어 dhcp로 IP 자동 할당
- 같은 네트워크의 다른 PC(예: 노트북)에서 접속 가능

---

## 🛠️ 트러블슈팅 & 💡 회고

### 👤 이정이
**문제**   : SSH 로그인 실패 로그가 리포트에 나오지 않음  
**원인**   : Ubuntu 24.04는 /var/log/auth.log 미생성 → journald 사용, 서비스명은 sshd로 기록됨  
**해결**   : journalctl -u ssh -u sshd -S today -U tomorrow --no-pager | grep "Failed password" 로 수정    
**회고**   : 배포판/버전에 따라 로그 경로와 유닛명이 다르므로 환경 의존성을 최소화한 스크립트 작성이 필요함  

---

### 👤 신기범
**문제**   : 스크립트 실행 시 출력이 끊여 가독성이 떨어짐  
**원인**   : tail 명령어는 기본적으로 파일의 마지막 10줄을 출력  
**해결**   : tail -n 0 -f 옵션으로 새로 append되는 log만 출력  
**회고**   : 로그 파일을 시각화 하는 도구 사용을 못 해서 아쉽고 로그 정보에 비해 유의미한 데이터 분석이 부족

---

## 📊 로그 흐름 구조 (Mermaid Diagram)

```mermaid
flowchart TD
    %% Inputs
    J["journalctl (ssh/sshd) - SSH 로그인 실패 로그"]:::src
    N["Nginx access.log - IP별 요청수 - URL/Status 등"]:::src

    %% Orchestrator
    S["nginx_daily_report.sh - 로그 수집/집계/요약"]:::proc

    %% Output
    O["analysis_append.log - 일자별 결과 누적 저장"]:::file

    %% Optional trigger
    C[(cron)]:::etc --> S

    %% Flow
    J --> S
    N --> S
    S --> O
    O --> V["결과 확인/리뷰"]:::view

    classDef src fill:#eef,stroke:#89a,stroke-width:1px;
    classDef proc fill:#efe,stroke:#6a6,stroke-width:1px;
    classDef file fill:#ffe,stroke:#caa,stroke-width:1px;
    classDef view fill:#f0f0f0,stroke:#aaa,stroke-width:1px;
    classDef etc fill:#ddd,stroke:#999,stroke-width:1px;
