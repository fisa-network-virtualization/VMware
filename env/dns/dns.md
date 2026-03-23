# DNS 설정
**Domain Name System(DNS)** 은 사람이 이해하기 쉬운 도메인 이름을 IP 주소로 변환해주는 인터넷의 전화번호부이다. 이를 기반으로 각 시스템이 안정적으로 서로를 식별하고 통신할 수 있도록 DNS 환경을 구성하는 것이 본 설정의 핵심이다.

## 1. 목적

vCenter 및 인프라 서비스들이 IP가 아닌 **FQDN**(Fully Qualified Domain Name)으로 상호 통신할 수 있도록 **DNS 서버**를 구성한다. <br/>
본 프로젝트에서 DNS 서버는 Alpine Linux(BIND)와 Windows Server 2022 두 가지 환경으로 구성하였다.

### DNS 서버가 필요한 이유
| 이유 | 설명 | 
|-----|-------|
| **FQDN 기반 통신** | vCenter 내부 서비스들은 IP가 아닌 FQDN으로 통신하도록 설계되어 있어 DNS 등록이 필수 |
| **인증서 발급** | vCenter 설치 시 FQDN 기준으로 인증서를 발급하며, 브라우저 접속 및 ESXi 호스트 등록에 해당 인증서를 사용 |


## 2. 구성도 / 개념

### 구성 개념
- DNS 서버는 Alpine Linux(BIND) 와 Windows Server 2022 두 가지 환경으로 이중 구성
- temu.finance 도메인에 대한 A 레코드를 관리하여 인프라 서비스 간 FQDN 기반 통신 지원
- vCenter는 내부 서비스 통신 및 인증서 발급 시 FQDN을 사용하므로 DNS 등록이 필수
- DNS 서버 자신도 127.0.0.1을 nameserver로 설정하여 자기 자신을 리졸버로 사용

### 전체 구조

<img width="868" height="635" alt="image" src="https://github.com/user-attachments/assets/d141f3ff-d031-492a-8182-257a6304f3bc" />

### 전체 흐름

- 클라이언트(ESXi / VM / vCenter) → DNS 서버에 FQDN 질의
- DNS 서버 → A 레코드 조회 후 IP 반환
- vCenter 설치 및 인증서 발급 → FQDN(vcsa.temu.finance) 기준으로 처리
- NTP 클라이언트(chrony) → ntp.temu.finance 조회 후 NTP 서버와 시간 동기화


## 3. 사전 준비사항, 설치, 설정 절차

### 3-A. Alpine Linux(BIND) DNS 서버

### 3-A-1. Alpine Linux 초기 설정
- 초기 설치 과정에서 아래의 항목들을 입력한다.

| 항목 | 값 | 
|-----|-------|
| ISO | alpine-standard |
| Hostname | dns-01 |
| Interfacee | th0 | 
| Domain | yellow.com |
| Password | VMware1! |
  
<img width="800" alt="linux_init" src="https://github.com/user-attachments/assets/ea5e92e0-30e5-44e8-95f7-b3456d633a9a" />

<img width="800" alt="image" src="https://github.com/user-attachments/assets/e3e39b81-9232-486c-98d2-79297453e27c" />

<img width="741" height="548" alt="image" src="https://github.com/user-attachments/assets/1fbec523-9ac3-425b-8ba6-d12fc03d5354" />

>
> 설치 중 inter-PG에 임시 연결하여 네트워크를 구성하고, 설치 완료 후 원래 포트그룹으로 복구한다.
>

```bash
# setup-alpine 실행 후 아래 항목 입력
Which one do you want to initialize? : eth0
IP address for eth0                  : dhcp
Manual network configuration?        : n
Mirror 사이트 선택                    : f
Which disk(s) would you like to use? : sda
How would you like to use it?        : sys
Erase disk and continue?             : y
```

설정 완료 후 재부팅한다.

---

### 3-A-2. DNS 정방향 조회 영역 파일 작성
> 문자로 된 이름(호스트명)을 어떤 IP 주소로 연결할지 정의

- `/var/bind/tenu.finance.zone` -> 아래의 내용을 추가한다.

```bash
$TTL 86400
@   IN  SOA  dns-01.temu.finance. admin.temu.finance. (
        2026031001 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

    IN  NS   dns-01.temu.finance.

dns-01  IN  A    172.20.100.30
ntp     IN  A    172.20.100.20
vcsa    IN  A    172.20.100.10
```

- 변경 후 재시작한다.

```bash
rc-service named restart
```

- Zone 파일 항목 설명

| 항목 | 값 | 
|-----|-------|
| $TTL | 레코드 기본 캐시 유지 시간(초) | 
| SOA | zone의 권한 정보 및 갱신 주기 정의 |
| Serial | zone 파일 버전 번호. 변경 시 반드시 증가시켜야 Secondary DNS가 갱신을 감지함 | 
| Refresh | Secondary DNS가 Primary에 갱신 여부를 확인하는 주기 | 
| Retry | 갱신 실패 시 재시도 주기 |
| Expire | 갱신 불가 시 Secondary DNS가 zone 데이터를 폐기하는 기간 |
| NS | 해당 zone의 네임서버 지정 |
| A | 호스트명 → IPv4 매핑 |

---

### 3-A-3. BIND Zone 설정
> 앞서 작성한 파일(DNS 정방향 조회 영역 파일)을 읽어서 서비스를 시작하도록 연결

- `/etc/bin/named.conf` -> 아래의 내용을 추가한다.

```bash
zone "temu.finance" IN {
    type master;
    file "temu.finance.zone";
};
```

<img width="358" height="212" alt="image" src="https://github.com/user-attachments/assets/61c316cc-d2b5-471b-85a7-1aa6ba72970c" />

- 변경 후 재시작한다.

```bash
rc-service named restart
```

### 3-A-5. NTP 클라이언트 설정
> 시간 동기화를 담당하는 Chrony 데몬의 메인 설정. 기준이 되는 서버로부터 시간을 받아오도록 설정
- `/etc/chrony/chrony.conf` -> 아래의 내용을 추가한다.
```bash
server ntp.temu.finance iburst
```
- `server`: 시간을 받아올 기준 서버(NTP 서버)를 지정
- `iburst`: 'Initial Burst'의 약자로, 서비스를 시작할 때 서버에 짧은 간격으로 8개의 패킷을 보내 시간 동기화를 아주 빠르게 완료하라는 옵션

- 변경 후 재시작한다.
```bash
rc-service chronyd restart
```

---

### 3-A-4. DNS resolv.conf 설정
> 사용하고자 하는 네임서버를 지정

- `/etc/resolv.conf` -> 아래의 내용을 추가한다.
```bash
nameserver 127.0.0.1
```

- 변경 후 재시작한다.
```bash
rc-service networking restart
```

---

### 3-B. Windows Server 2022 DNS 서버

### 3-B-1. DNS 역할 추가
- 서버 관리자 -> 역할 및 기능 추가 -> 서버 역할 -> DNS 서버 체크 -> 설치

<img width="919" height="615" alt="image" src="https://github.com/user-attachments/assets/1c764d6e-c916-4ae4-9284-b66b296acd56" />

---

### 3-B-2. 정방향 조회 영역 추가
- 서버 관리자 -> 도구 -> DNS -> 정방향 조회 영역 우클릭 -> 새 영역 추가
- 영역 이름 입력 (예: temu.finance)

<img width="1232" height="594" alt="image" src="https://github.com/user-attachments/assets/ef6ccf4d-9b4f-450f-b361-094784b7528f" />

---

### 3-B-3. 호스트(A) 레코드 추가
- 생성한 영역에서 우클릭 -> 새 호스트(A 또는 AAAA) 선택 후 아래의 항목을 추가

| 호스트 이름 | IP 주소 |
| -------- | -------- |
| vcsa | 172.20.100.10 |
| ntp | 172.20.100.20 |


<img width="895" height="453" alt="image" src="https://github.com/user-attachments/assets/5d9a613d-740c-43d5-bee3-2ee18e8738fa" />


## 4. 확인 / 검증 방법

### DNS 조회 테스트 (Alpine Linux)

```bash
# NTP 서버 도메인 조회
nslookup ntp.temu.finance

# vCenter 도메인 조회
nslookup vcsa.temu.finance

# DNS 서버 자신 조회
nslookup dns-01.temu.finance


**정상 응답 예시**

Server:    127.0.0.1
Address:   127.0.0.1#53

Name:      ntp.temu.finance
Address:   172.20.100.20
```


<img width="702" height="303" alt="image" src="https://github.com/user-attachments/assets/557afc3d-5d44-409f-b0d3-b67d50b1e510" />



## 5. 참고사항

- Serial 번호 관리 — Zone 파일을 수정할 때마다 Serial을 반드시 증가시켜야 한다. 일반적으로 YYYYMMDDNN 형식(예: 2026031001)을 사용하며, 같은 날 두 번째 수정이면 2026031002로 올린다.
- Alpine Linux 네트워크 임시 연결 — 초기 설치 시 패키지 다운로드를 위해 `inter-PG`에 임시 연결 후 설치 완료 후 반드시 원래 포트그룹으로 복구해야 한다.
- iburst 옵션 — chrony 설정의 iburst는 서버 시작 시 빠르게 시간을 동기화하기 위해 초기에 여러 패킷을 연속 전송하는 옵션이다.
- Windows vs Linux DNS — 본 실습에서 두 DNS 서버는 동일한 도메인을 서비스하므로 운영 환경에서는 하나를 Primary, 다른 하나를 Secondary로 구성해 이중화한다.

