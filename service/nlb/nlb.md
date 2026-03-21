# Nginx 기반 로드밸런서 구축

## 1. 목적

NFS 미션에서 각 웹 서버는 Shared Storage(NFS)를 이용하여 동일한 `index.html`을 마운트받고,  
스토리지에서 파일을 수정하면 모든 웹 서버에 동시에 반영되도록 구성하였다.

즉, 이 미션의 핵심 목적은 다음과 같다.

- 동일한 웹 서버를 여러 대 구성한다.
- 사용자가 어느 서버에 접속하더라도 같은 서비스에 접속한 것처럼 느끼게 한다.
- 특정 서버에 사용자가 과도하게 몰려 서버가 다운되는 상황을 방지한다.
- 여러 서버로 트래픽을 분산하여 전체 서비스의 안정성과 가용성을 높인다.

따라서 본 작업에서는 동일한 웹 서버 여러 대를 대상으로 트래픽을 분산하기 위해  
**Nginx 기반 로드밸런서(Load Balancer)** 를 구축하였다.



## 2. 구성도 / 개념

### 2-1. 전체 구조

![로드밸런서 구조도](./images/nlb-architecture.png)

### 2-2. 구성 개념

- 웹 서버 VM 3대는 모두 동일한 콘텐츠를 제공하도록 구성됨
- 각 웹 서버는 외부 접근이 가능해야 하므로 **DVS의 Inter-PG** 에 연결
- 로드밸런서는 **Nginx** 를 사용하여 구성
- 외부 사용자의 요청은 먼저 pfSense를 통과한 뒤, 로드밸런서 VM으로 전달됨
- 로드밸런서 VM은 전달받은 요청을 다시 내부 웹 서버 3대로 분산함

### 2-3. 로드밸런서 구조 선정 이유

로드밸런싱 방식으로 다음 두 가지를 고려하였다.

- pfSense 방화벽을 이용한 로드밸런싱
- Nginx를 이용한 로드밸런싱

#### 선택 기준

##### 1) L4와 L7의 차이
- pfSense는 네트워크 계층(L4)에서 IP와 포트 기반으로 트래픽을 분산
- Nginx는 애플리케이션 계층(L7)에서 URL, 쿠키, HTTP 헤더 등을 보고 더 정교하게 분산 가능

##### 2) 보안과 장애 격리
- 방화벽은 외부 공격을 가장 먼저 받는 핵심 보안 장비
- 여기에 로드밸런싱 역할까지 부여하면, 트래픽 폭주 시 방화벽 자체 장애로 이어질 수 있음
- 따라서 방화벽과 로드밸런서를 분리하는 것이 안정적임

##### 3) 비용
- 가상화 환경에서는 Nginx VM 하나를 추가로 올리는 비용이 크지 않음

##### 4) 설정 편의성
- pfSense는 NAT와 방화벽 규칙을 함께 조정해야 함
- Nginx는 설정 파일 중심으로 빠르게 구성 가능

#### 결론
- 방화벽은 **보안 및 포워딩 역할**
- 웹 트래픽 분산은 **Nginx 기반 전문 프록시 서버**가 담당

### 2-4. 트래픽 흐름

기존 웹 서버 접근 흐름:

```text
인터넷 → 외부망 스위치 → 방화벽 → 내부망 스위치 → ESXi → DVS → 웹 서버 VM
```

로드밸런서 적용 후 흐름:
```text
인터넷 → 외부망 스위치 → 방화벽 → 내부망 스위치 → ESXi → DVS → 로드밸런서 VM → 웹 서버 1, 2, 3
```

### 2-5. One-Arm Proxy 구조
로드밸런서 VM은 내부망에 위치하며,
외부 트래픽을 pfSense가 로드밸런서 VM으로 전달하고,
로드밸런서 VM이 다시 같은 내부 네트워크 상의 웹 서버들로 트래픽을 분산하는
One-Arm Proxy 구조로 동작한다.

## 3. 사전 준비사항, 설치 또는 설정 절차
### 3-1. 사전 준비사항
#### 웹 서버 VM 구성 정보
- 웹 서버 1
    - OS: Rocky Linux Minimal
    - IP: 10.40.110.21/24
    - Gateway: 10.40.110.1
    - DVS 연결 포트그룹: Inter-PG
    - 적재 위치: ESXi2 (172.20.100.102)
    - Python 웹서버 동작
- 웹 서버 2
    - OS: Rocky Linux Minimal
    - IP: 10.40.110.22/24
    - Gateway: 10.40.110.1
    - DVS 연결 포트그룹: Inter-PG
    - 적재 위치: ESXi3 (172.20.100.103)
    - Python 웹서버 동작
- 웹 서버 3
    - OS: Rocky Linux Minimal
    - IP: 10.40.110.23/24
    - Gateway: 10.40.110.1
    - DVS 연결 포트그룹: Inter-PG
    - 적재 위치: ESXi2 (172.20.100.104)
    - Python 웹서버 동작

> 위 3대의 웹 서버는 NFS 미션에서 동일한 콘텐츠를 제공하도록 이미 구성되어 있어야 한다.

### 3-2. 로드밸런서 VM 생성

로드밸런싱은 Nginx 를 이용하여 구성하였다.

#### 로드밸런서 VM 사양
- vCPU: 1 Core
- RAM: 1GB
- Disk: 16GB (Thin Provisioning)
- 네트워크 어댑터: VMXNET3
- OS: Rocky Linux Minimal
- IP: 10.40.110.30/24
- Gateway: 10.40.110.1
- DVS 연결 포트그룹: Inter-PG
- 적재 위치: ESXi3 (172.20.100.103)

> 구성도 상으로는 ESXi2에 위치해 보일 수 있으나, 실제 적재 위치는 ESXi3이다.

### 3-3. 로드밸런서 VM 네트워크 설정
#### 1) IP 할당
![로드밸런서 구조도](./images/nlb-architecture.png)

```bash
sudo nmtui
Edit a connection → ens192 에서 IP와 게이트웨이 설정
```
#### 2) 네트워크 재시작
```bash
sudo systemctl restart NetworkManager
```

#### 3) 네트워크 설정 확인
```bash
ip a
```

### 3-4. Nginx 설치
```bash
sudo dnf install epel-release -y
sudo dnf install nginx -y

sudo systemctl enable --now nginx
sudo systemctl status nginx
```

- epel-release 설치
- Nginx 설치
- 서비스 시작 및 부팅 시 자동 실행 설정
- 상태 확인

### 3-5. Rocky Linux 방화벽 설정
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

- HTTP(80), HTTPS(443) 통신 허용
- 방화벽 설정 적용 후 확인

### 3-6. Nginx 로드밸런싱 설정
#### 1) 테스트용 SSL 인증서 생성

HTTPS 접속을 위한 테스트용 인증서를 생성한다.

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/nginx/nginx-selfsigned.key \
-out /etc/nginx/nginx-selfsigned.crt
```
![로드밸런서 구조도](./images/nlb-architecture.png)

> 명령어 실행 후 표시되는 입력 항목은 기본값으로 진행 가능

#### 2) Nginx 설정 파일 열기
```bash
sudo vi /etc/nginx/nginx.conf
```

#### 3) Nginx 설정 파일 수정
```bash
http {
    # ... 기존 설정 유지 ...

    # 1. 분산 대상 웹 서버 그룹
    upstream my_web_servers {
        server 10.40.110.21:80;
        server 10.40.110.22:80;
        server 10.40.110.23:80;
    }

    # 2. HTTP 요청을 HTTPS로 리다이렉트
    server {
        listen 80;
        listen [::]:80;
        server_name _;

        return 301 https://$host$request_uri;
    }

    # 3. HTTPS(443) 설정
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name _;

        ssl_certificate /etc/nginx/nginx-selfsigned.crt;
        ssl_certificate_key /etc/nginx/nginx-selfsigned.key;

        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 10m;
        ssl_ciphers PROFILE=SYSTEM;
        ssl_prefer_server_ciphers on;

        location / {
            proxy_pass http://my_web_servers;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

#### 4) 문법 검사 및 설정 적용
```bash
sudo nginx -t
sudo systemctl reload nginx
```

- nginx -t : 문법 검사
- reload : 변경된 설정 반영

#### 5) SELinux 허용 설정

Rocky Linux의 SELinux는 기본적으로 Nginx가 다른 서버로 접속하는 것을 차단할 수 있으므로,
프록시 동작을 위해 아래 설정이 필요하다.
```bash
sudo setsebool -P httpd_can_network_connect 1
```

### 3-7. pfSense NAT 설정
![로드밸런서 구조도](./images/nlb-architecture.png)

외부에서 들어오는 HTTPS(443) 요청을 로드밸런서 VM으로 전달하도록 NAT를 수정한다.

#### 설정 내용
- 경로: Firewall → NAT → Port Forward
- WAN Address로 들어오는 TCP 443 트래픽을
- 로드밸런서 VM (10.40.110.30) 으로 전달
- 해당 트래픽을 허용하도록 설정

## 4. 확인 / 검증 방법
### 4-1. 각 웹 서버 실행

각 웹 서버에서 Python 웹서버를 실행한다.

```bash
sudo python3 -m http.server 80
```

### 4-2. 외부 접속 확인
- WAN Address인 192.168.0.22 로 접속
- HTTPS를 통해 서비스 접속 확인

### 4-3. 로드밸런싱 동작 확인

페이지를 여러 번 새로고침하면서 각 웹 서버의 로그를 확인한다.

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

또는 각 웹 서버 측에서 접속 로그를 확인하여
요청이 특정 서버에 몰리지 않고 균등하게 분산되는지 검증한다.

#### 기대 결과
![로드밸런서 구조도](./images/nlb-architecture.png)
- 사용자 입장에서는 동일한 웹 페이지에 접속하는 것처럼 보임
- 실제 요청은 웹 서버 1, 2, 3으로 분산됨

## 5. 트러블슈팅
### 5-1. 502 Bad Gateway 발생
#### 오류 내용
![로드밸런서 구조도](./images/nlb-architecture.png)
- 웹 서버 접속은 되지만 502 Bad Gateway 가 발생함
- 아래 명령어로 오류 로그 확인 가능

```bash
sudo tail -n 20 /var/log/nginx/error.log
```

#### 확인된 원인
- 113: No route to host (10.40.110.21, 10.40.110.23)
- 웹 서버 쪽 Rocky Linux 자체 방화벽이 80번 포트를 허용하지 않음

#### 해결 방법

각 웹 서버 VM에서 80번 포트 허용:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```

### 5-2. 웹 서버 2번(10.40.110.22)으로만 접속이 되지 않는 경우
#### 오류 내용
- 2번 웹 서버에 대해서만 로드밸런싱이 되지 않음
- Nginx 에러 로그에서 110: Connection refused 발생
- IP까지는 도달하지만 해당 서버에서 80번 포트를 정상적으로 서비스하지 않음

#### 원인
- 동일한 IP 주소를 가진 다른 VM이 존재함
- IP 충돌로 인해 정상적인 요청 전달이 불가능했음

#### 해결 방법
- 중복된 IP를 사용하는 VM을 확인
- 해당 VM의 IP 주소를 변경하여 문제 해결

## 6. 참고사항
- 본 로드밸런서는 NFS 기반 동일 콘텐츠 웹 서버를 전제로 구성됨
- pfSense는 보안 및 트래픽 포워딩, Nginx는 웹 트래픽 분산 역할로 분리하여 설계함
- Nginx는 L7 로드밸런서로 URL, 헤더 등 애플리케이션 레벨 확장이 가능함
- 테스트 환경에서는 자체 서명(Self-Signed) 인증서를 사용했으나, 운영 환경에서는 공인 인증서 사용이 권장됨
- 이미지 파일은 프로젝트 구조에 맞춰 ./images/ 폴더 하위에 배치해야 함
- 웹 서버 및 로드밸런서의 방화벽 설정, NAT 설정, SELinux 설정이 모두 맞아야 정상 동작함