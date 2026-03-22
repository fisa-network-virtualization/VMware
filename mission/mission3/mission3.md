# vCenter 백업



## 백업 대상으로 TrueNAS(SMB)를 사용하는 이유

vCenter 파일 기반 백업은 SMB, SFTP, FTP, NFS 등의 네트워크 경로를 백업 위치로 지정해야 한다.

- **클러스터 외부 저장**: 백업 데이터가 vSAN 클러스터 밖에 저장되어, 클러스터 전체 장애 시에도 백업이 안전하게 보존된다.
- **ZFS 안정성**: TrueNAS의 ZFS는 데이터 무결성 검증(체크섬)을 기본 제공하여 백업 파일의 손상을 방지한다.
- **추가 비용 없음**: 기존에 운영 중인 TrueNAS를 활용한다.

---

## 1. TrueNAS에 백업을 위한 Dataset 생성 (SMB)


### ① TrueNAS SMB 서비스 허용 계정 생성

TrueNAS에서 SMB 접근이 가능한 계정을 생성한다(`truenas_admin`).  
vCenter가 백업 파일을 쓸 때 인증에 사용되므로, 해당 Dataset에 대한 쓰기 권한이 필요하다.

### ② Dataset 생성

TrueNAS 웹 UI → **Datasets** 메뉴에서 새 Dataset을 생성한다.

<img width="2550" height="1353" alt="image" src="https://github.com/user-attachments/assets/7bcd92d2-40d6-4e28-89f4-365acee8ee97" />


### ③ SMB 공유 설정

생성한 Dataset을 SMB로 공유한다.

| 항목 | 설정값 |
|------|--------|
| 공유 이름 | `vcsa-backup` |
| 공유 경로 | `smb://172.20.100.88/vcsa-backup` |

---

## 2. vCenter VAMI 접속 및 백업 설정

`https://172.20.100.10:5480`으로 접속한다.  
VAMI는 vCenter Appliance의 관리 전용 인터페이스로, 백업/복원, 서비스 관리, 업데이트 등 어플라이언스 수준의 설정을 담당한다.

### ① VMware PSC Health 서비스 시작

서비스 메뉴에서 **VMware PSC Health**가 중지되어 있다면 시작한다.
<img width="1655" height="512" alt="image (1)" src="https://github.com/user-attachments/assets/67aa3441-0621-41f1-9ed5-8ca8386bb77b" />


### ② 백업 일정 생성 및 즉시 백업
<img width="1308" height="953" alt="image (2)" src="https://github.com/user-attachments/assets/9bfdfedc-0d2c-4a75-88fa-65c2c3296b8c" />


### ③ 백업 결과 확인

백업 완료 후 VAMI 백업 메뉴 및 TrueNAS의 `vcsa-backup` Dataset에서 백업 파일이 정상적으로 생성되었는지 확인한다.
<img width="1690" height="697" alt="image (3)" src="https://github.com/user-attachments/assets/31e8a0c8-faba-4d56-af3e-7f746bbb64e5" />
