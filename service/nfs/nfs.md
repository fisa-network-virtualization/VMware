# NFS 공유 저장소 구축

## 1. 목적

- 다중 웹 서버 환경에서 웹 콘텐츠의 일관성을 유지하고 효율적으로 관리하기 위해 NFS(Network File System)를 활용한 공유 저장소 체계를 구축
- 여러 대의 웹 서버나 애플리케이션 서버가 동일한 데이터에 접근해야 할 때, 각 서버마다 데이터를 복사해 둘 필요 없이 NFS 서버 한 곳에만 저장해 두고 필요한 VM 모두 마운트하여 실시간으로 읽고 쓸 수 있음

<br/>

## 2. 구성도 / 개념

### NFS Server 선택지 비교

3개의 Web VM이 동일한 웹 루트 directory를 바라보게 하려면 주로 NFS(Network File System) 공유를 사용함.
TrueNAS에 직접 할당할 지, 별도의 Linux VM을 띄워서 할당할 지 선택 가능.

|  | TrueNAS | 별도의 VM |
|---|---|---|
| 장점 | 데이터 안정성 (TrueNAS의 ZFS 파일 시스템), 리소스 효율성 | 높은 자유도 |
| 단점 | 네트워크 의존도 높음 (스토리지 네트워크 대역폭 확보) | 이중 오버헤드, 불필요한 리소스 낭비 가능성, 관리 부담 증가 |

<br/>

## 3. 사전 준비사항, 설치 또는 설정 절차

### 3-1. NFS Server 설정 (TrueNAS 활용)

**① TrueNAS에서 ZFS 데이터셋(공유 폴더) 생성**

- Storage → Create Pool
  - 하드디스크가 1개 → Stripe Layout, 2개 → Mirror

**② NFS 공유 활성화 및 권한 설정**

- Shares → UNIX Shares → Add
- 앞서 만든 데이터셋 경로 선택 (`/mnt/pool_name/web_data`)
- Authorized Networks → Storage-PG 서브넷 대역 입력 (`10.40.110.0`)
- Submit 후 NFS 서비스 활성화 Enable

### 3-2. NFS Client 설정

**① NFS 패키지 설치**

```bash
sudo dnf install nfs-utils -y
```

**② 클라이언트 VM에서 마운트**

```bash
sudo mount -t nfs 172.16.30.88:/mnt/web_data/files /mnt/web_data/files
                       ↑                                  ↑
               [trueNAS 서버_IP]                    [내 디렉토리 경로]
```

**③ 개별 Rocky Linux 방화벽 포트 허용 (기존 443, 로드밸런서 적용 후 80)**

```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=80/tcp

sudo firewall-cmd --reload

sudo firewall-cmd --list-all
# 여기서 443/tcp 뜨는지 확인해야 함

# index.html 있는 폴더에서 실행할 것
sudo python3 -m http.server 443
sudo python3 -m http.server 80
```

<br/>

## 4. 확인 / 검증 방법

- 정상 마운트 여부 확인

```bash
df -h  # 또는 mount 명령어로 마운트 목록 확인
```

<br/>

## 5. 참고사항

### 기존에 참고했던 코드

**NFS Server 설정 (별도 Linux VM 방식)**

```bash
# NFS 패키지 설치
sudo dnf install nfs-utils -y
# 서비스 시작 및 활성화
sudo systemctl enable --now nfs-server

# 공유할 디렉토리 생성 및 권한 설정
sudo mkdir -p /mnt/nfs_share
sudo chmod -R 777 /mnt/nfs_share

# 접근 권한을 줄 네트워크 대역 설정
sudo vi /etc/exports
# ㄴ> /mnt/nfs_share 172.16.30.0/24(rw,sync,no_root_squash) 작성

# 설정 내용 적용
sudo exportfs -arv

# NFS 및 관련 서비스 허용
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd

# 방화벽 설정 새로고침하여 규칙 적용
sudo firewall-cmd --reload
```

**리눅스 IP 설정하기**

```bash
sudo nmcli con mod [인터페이스명] ipv4.addresses 172.16.30.x/24
sudo nmcli con mod [인터페이스명] ipv4.gateway 172.16.30.x
sudo nmcli con mod [인터페이스명] ipv4.dns "172.16.30.77"
sudo nmcli con mod [인터페이스명] ipv4.method manual

sudo nmcli con up [인터페이스명]  # 저장 명령어, 반드시 필요

# 인터넷 연결 위해 DHCP로부터 IP 할당 받기
sudo nmcli con mod [인터페이스명] ipv4.method auto

sudo nmcli con mod [인터페이스명] ipv4.addresses ""
sudo nmcli con mod [인터페이스명] ipv4.gateway ""
sudo nmcli con mod [인터페이스명] ipv4.dns ""

sudo nmcli con up [인터페이스명]
```