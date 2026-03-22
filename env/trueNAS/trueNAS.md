# TrueNAS Install

## 1. 목적

여러 대의 ESXi host와 그 위에 VM이 올라간 형태로 운영을 하기 위해선, 여러 ESXi host들이 모두 공유하는 스토리지를 설정하고 관리할 수 있는 OS를 설치해야 한다.\
단순히 NFS를 사용하려고 한다면 간단한 Linux VM으로도 사용할 수 있겠지만, 배정된 저장공간을 효율적으로 사용하기 위해선 스토리지 관리 전용 OS가 필수적이다.

즉, 핵심 목적은 다음과 같다.

- ESXi host들에 연결된 공용 스토리지를 효율적으로 관리한다.
- iSCSI, NFS와 같은 스토리지 공유 설정을 진행한다.


따라서 본 문서에서는 TrueNAS를 설치하는 과정, 그리고 그 설치 과정에서 필요한 작업들을 정리하였다.



## 2. 구성도 / 개념

### 2-1. 구성 개념

구성도 내에 있는 모든 VM들이 공유 스토리지를 사용하기 위해 TrueNAS를 설치하는 것이므로, 다음과 같은 설정이 필요하다.

- TrueNAS에서 쓸 수 있도록 스토리지 추가 배정
- 내부 VM들이 공유 스토리지 기능을 사용할 수 있도록 PG 설정


### 2-2. IP, PG 2개 할당

Cluster 내부에서 사용할 수 있는 PG와 외부 VM들이 사용하는 PG의 IP가 서로 다름\
→ TrueNAS에 2개의 IP와 2개의 PG가 할당되어야 함

## 3. 사전 준비사항, 설치 또는 설정 절차
### 3-1. 사전 준비사항
#### TrueNAS VM 설정

- CPU: 4
- RAM: 16GB
- Storage: 32GB
- OS: Debian Linux 4
- IP 1: 172.20.100.88/24
- IP 2: 10.40.130.10/24

> vSwitch0의 Storage-PG와 Managed-PG에 모두 연결하여 사용할 수 있도록 다음과 같이 설정한다.

### 3-2. TrueNAS ISO 파일 마운트

설치해둔 데이터스토어 자체에 iso 파일을 올려 ESXi Host 클라이언트로 VM을 설치한다.

### 3-3. TrueNAS 설치

내부 설치 후, VM 내에 직접 접속하여 VM Network를 설정한다.
위에서 할당한 IP와 Gateway를 설정한 뒤, 관리용 IP를 사용하여 접속한다.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7441385f-facf-48e8-8132-ab6999c186f9" />


### 3-4. iSCSI 설정

VM들이 공유하는 스토리지를 만들기 위해 iSCSI 설정을 진행한다.

#### 1) 추가 하드디스크 용량 할당

<img width="1002" height="744" alt="image" src="https://github.com/user-attachments/assets/642d5b54-21c4-4480-ba20-4eed56bcb71a" />


650GB의 용량을 TrueNAS VM에 할당한다.\
이렇게 되면, TrueNAS에 650GB의 추가 디스크가 꽂힌 것으로 판정된다.
#### 2) 추가된 하드디스크 등록

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9dbbea4c-b737-4fdd-a65c-859d4a383dfa" />


TrueNAS GUI에서 할당받은 디스크를 추가한다.
#### 3) Zvol 추가

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/322f9f42-48dc-4815-9aa7-c3c7e41ac272" />


할당받은 하드디스크 용량을 iSCSI로 활용하기 위해 필수적인 설정이다.

#### 4) iSCSI Share

<img width="820" height="644" alt="image" src="https://github.com/user-attachments/assets/a5e4c5ee-e340-4929-9683-d0191cce200e" />


iSCSI 서비스를 실행한다.

#### 5) Network Configuration


<img width="1249" height="776" alt="image" src="https://github.com/user-attachments/assets/6fe37a2e-9d47-4db4-85fb-16790b527aa4" />


가동 중인 iSCSI를 VM들에서 사용할 수 있도록 ESXi 호스트 클라이언트에서 동적 대상에 Storage-PG에 할당된 IP인 10.40.130.10을 등록한다.


## 4. 확인 / 검증 방법
### 4-1. VCSA Installation

<img width="1394" height="1020" alt="image" src="https://github.com/user-attachments/assets/07f642f6-37f5-4fbb-8b5a-7ba5af8767da" />


vCenter 설치 시 iSCSI Datastore가 정상적으로 잡히는 것을 확인할 수 있다.


### 4-2. Adding New VMs

추가적인 VM 설치의 경우 iSCSI를 활용하여 효과적으로 저장소를 관리할 수 있다.


## 5. 트러블슈팅

없음

## 6. 참고사항
- [TrueNAS 가이드](https://svrforum.com/nas/2516465)
- 설치 시 모든 VM에서 Thin Provisioning을 활성화하였음
- 실제 구동 환경에서 똑같이 600GB만을 할당할 경우, VM들의 스토리지 부족 현상이 발생할 수 있음
- NAS를 구성하는 것을 기본 전제 조건으로 만들어진 OS이므로, 이 문서처럼 단일 스토리지를 구성할 경우 경고 발생

