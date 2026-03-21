# NFS 기반 Web Server 동기화

다중 웹 서버 환경에서 웹 콘텐츠의 일관성을 유지하고 효율적으로 관리하기 위해 **NFS**(Network File System)를 활용한 공유 저장소 체계를 구축하였다.

## 1. 목적

### 🟢 데이터의 중앙 집중화 및 실시간 공유
* **중복 제거:** 여러 대의 웹 서버(Web Server)나 애플리케이션 서버가 동일한 데이터에 접근해야 할 때, 각 서버마다 데이터를 복사할 필요가 없음
* **실시간 동기화:** NFS 서버 한 곳에 데이터를 저장하고 필요한 모든 VM에 마운트하여, 실시간으로 동일한 데이터를 읽고 쓸 수 있는 환경을 구축

---

## 2. 구성도 / 개념

> **목표:** 3개의 Web VM이 동일한 웹 루트 디렉토리(Web Root Directory)를 바라보게 하여 데이터 일관성을 유지

### 스토리지 구성 방식 비교
TrueNAS를 직접 활용할지, 별도의 Linux VM을 파일 서버로 운영할지에 대한 비교 분석

| 구분 | TrueNAS (직접 할당) | File Server VM (Linux) |
| :--- | :--- | :--- |
| **장점** | **데이터 안정성:** ZFS 파일 시스템 활용 가능<br>**리소스 효율:** 별도 OS 레이어 없음 | **높은 자유도:** 사용자 정의 설정 및 패키지 설치 용이 |
| **단점** | **네트워크 의존성:** 스토리지 네트워크 대역폭 확보 필수 | **오버헤드:** 이중 가상화로 인한 리소스 낭비<br>**관리 부담:** OS 유지보수 포인트 증가 |

> 본 프로젝트에서는 TrueNAS를 활용하여 NFS를 구성하는 방식을 선택함

### 구성도
> 수정 예정

1. **Storage:** TrueNAS / ZFS Pool 구성
2. **Protocol:** NFSv4 서비스 활성화
3. **Clients:** Web Server #1, #2, #3 (`/var/www/html` 마운트)

### 흐름
> 수정 예정

---

## 3. 사전 준비사항, 설치, 설정 절차

### 3-1. 사전 준비 사항

**NFS Server 설정**

1. TrueNAS에서 ZFS 데이터셋(공유 폴더) 생성
<img width="800" alt="zfs_create" src="https://github.com/user-attachments/assets/eab2b0c4-1d97-4a4d-b3e3-7394d62a02e0" />

> Storage > Create Pool 클릭하여 생성.
하드디스크가 1개 → Stripe Layout, 2개 → Mirror 선택

- 생성된 ZFS 데이터셋 (`web_data`)
<img width="800" alt="create_pool" src="https://github.com/user-attachments/assets/3c907d2e-a6a1-44f5-bb3b-e0fe7da5c083" />


2. NFS 공유 활성화 및 권한 설정
<img width="800" alt="nfs_active" src="https://github.com/user-attachments/assets/45fbb95b-5aae-44c9-ae1d-38065723fcde" />

- Shares > UNIX Shares > add 클릭
<img width="800" alt="add_unix_shares" src="https://github.com/user-attachments/assets/10c41c26-5d63-442a-9963-a14a42ba0205" />

- 앞서 만든 데이터셋 경로 선택 (`/mnt/pool_name/web_data`)
- Authorized Networks > Storage-PG 서브넷 대역 입력 (`10:40.110.0`)
<img width="240" alt="nfs_share_detail" src="https://github.com/user-attachments/assets/e1ba380e-c651-4ef8-9420-3a2c87e1145f" />

> Submit 후 NFS 서비스 활성화 Enable

