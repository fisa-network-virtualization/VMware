# VCSA Install

## 1. 목적

여러 대의 ESXi host와 그 위에 VM이 올라간 형태로 운영을 하기 위해선, 여러 ESXi host들을 한 번에 설정하고 관리할 수 있는 vCenter가 큰 도움이 된다.\
또한, 현재 구축하려는 인프라에는 **DVS(Distributed vSwitch)** 를 설치해야 하므로 vCenter 설정이 필수적이다.

즉, 핵심 목적은 다음과 같다.

- ESXi host들을 통틀어 관리한다.
- DVS를 활용하여 VM들을 통합적으로 관리할 수 있는 네트워크를 구성한다.
- vCenter에서 사용 가능한 HA, DRS 기능들을 설정한다.

따라서 본 문서에서는 vCenter를 설치하는 과정, 그리고 그 설치 과정에서 필요한 작업들을 정리하였다.



## 2. 구성도 / 개념

### 2-1. 전체 구조

<img width="1687" height="1505" alt="image" src="https://github.com/user-attachments/assets/820169b4-d305-45ac-af53-d2ee632a823a" />


위의 VM들과 ESXi들을 모두 관리하는 형태로 구성한다.

### 2-2. 구성 개념

- ESXi Host 4개, DNS 서버 2개, NTP 서버 2개, pfSense, TrueNAS가 모두 한 개의 ESXi Host 위에 존재하는 **중첩된 가상화** 환경
- ESXi Host 4개 내에 모든 VM들을 넣고, ESXi Host 4개를 Cluster로 묶어서 DVS로 관리 예정
- 이 모든 과정을 관리하는 VCSA를 설치할 Host를 정하고, DNS 서버를 이용해 접속할 수 있도록 설정 예정


### 2-3. DNS 서버의 필요성

1. vCenter 내부의 여러 서비스들은 ip가 아닌 FQDN으로 통신하게 되어 있음\
    → vCenter의 FQDN이 DNS 서버에 등록되어 있어야 함

2. vCenter 설치 시 FQDN 기준으로 인증서를 발급\
    → 브라우저 접속, ESXi 호스트 등록 등에 인증서를 사용

## 3. 사전 준비사항, 설치 또는 설정 절차
### 3-1. 사전 준비사항
#### DNS 서버 구성 정보
- DNS 서버 1
    - OS: Rocky Linux Minimal
    - IP: 172.20.100.30/24
    - Gateway: 172.20.100.3
    - vSwitch 연결 포트그룹: VM Network
    - 적재 위치: 외부 ESXi (172.20.100.1)
    - FQDN: *.temu.finance
- DNS 서버 2
    - OS: Windows Server 202
    - IP: 172.20.100.31/24
    - Gateway: 172.20.100.3
    - vSwitch 연결 포트그룹: VM Network
    - 적재 위치: 외부 ESXi (172.20.100.1)
    - FQDN: *.temu.finance


#### iSCSI 데이터스토어 설정

- TrueNAS 설정으로 진행
    - 600GB 할당
    - IP: 10.40.130.10/24
    - vSwitch0의 Storage-PG 사용


> 위 2대의 DNS 서버와 iSCSI는 VCSA 설치 전에 이미 구성되어 있어야 한다.

### 3-2. VCSA iso 파일 마운트, 실행

설치 방식이 네트워크를 사용하여 ESXi 상에 올리는 방식이므로, ESXi와 같은 네트워크에 연결된 노트북에서 실행해도 된다.

### 3-3. VCSA Install - Stage 1
#### 1) IP 할당

<img width="1304" height="1037" alt="image" src="https://github.com/user-attachments/assets/5ace3f46-85ed-49e5-870f-0e02d0009182" />


vCenter를 관리하는 VM의 IP를 설정한다.
#### 2) VM 이름, root password 설정
<img width="1304" height="1037" alt="image" src="https://github.com/user-attachments/assets/ee0bbebe-30ef-44b3-939a-95bbc3a356a5" />


vCenter 자체를 관리하는 데 사용하는 root 계정의 비밀번호를 설정한다.
#### 3) Deployment Size Select

<img width="1304" height="1037" alt="image" src="https://github.com/user-attachments/assets/1c808838-dc0d-44eb-b69b-d916e395069f" />

Tiny로 설정하여 물리 서버에 맞는 형태로 운영 가능하도록 한다.

#### 4) Datastore Select

<img width="1394" height="1020" alt="image" src="https://github.com/user-attachments/assets/991503c6-215c-4baf-8bf5-c824d92362f8" />


ESXi 내부에 들어가 있는 iSCSI를 사용하도록 지정한다.

**Thin Disk Mode** 를 반드시 활성화해야 씬 프로비저닝이 활성화되어 579GB를 한번에 할당시키지 않고 나눠서 할당시켜 다른 VM도 사용할 수 있도록 자원을 관리할 수 있다.
#### 5) Network Configuration

<img width="1304" height="1037" alt="image" src="https://github.com/user-attachments/assets/00d42bca-c2d5-4be3-aeeb-94ca35e8beb0" />


- FQDN: vcsa.temu.finance. VCSA에 접속할 때 사용할 DNS 주소
- IP Address: 172.20.100.200. vCenter VM의 IP 주소
- Subnet mask: 24
- Gateway: 실제 서버 구성 시 사용하는 Gateway IP 입력, VCSA 설치 당시엔 게이트웨이가 없었으므로 그냥 IP주소와 똑같이 입력
- DNS Servers: 172.20.100.31. DNS 서버의 IP 주소


#### 6) vSwitch Security Configuration

<img width="895" height="635" alt="image" src="https://github.com/user-attachments/assets/86f5ad87-da57-4f04-a76d-271e9909c82a" />


Promiscuous mode, MAC 주소 변경, 위조 전송(Forged transmits)을 모두 활성화시켜야 정상적인 vCenter 환경 설정 가능

#### 7) Stage 1 Complete

설치 전 마지막 설정 확인 화면이 나오고, 계속하면 Stage 1이 완료된다.

이후 약 10분간의 설치 과정을 기다리면 Stage 2를 진행한다.\
Stage 2를 VCSA 설치 프로그램에서 진행할 수도 있고, vCenter 관리 VM의 IP주소:5480으로 접속하여 설치를 이어갈 수 있다.

### 3-4. VCSA Install - Stage 2

#### 1) SSO Configuration

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2a287380-205e-4843-bc90-1b4793b490db" />


- SSO Domain Name: vpshere.local - 사용자 계정들의 domain으로 사용할 주소 설정
- SSO Username: Administrator
- SSO Password: 관리자 계정에 접속 시 사용할 비밀번호


#### 2) Stage 2 Complete

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/1d7af7b9-515c-40ee-8e55-a28e26ffe413" />


다음과 같이 최종 설정이 보여지고, 이후 Stage 2까지 완료되면, vcsa.temu.finance 주소로 접속할 경우 vSphere client에 접속할 수 있다.

CEIP 설정은 원활한 테스트 환경 구성을 위해 거부하였음.



## 4. 확인 / 검증 방법
### 4-1. vSphere Client

<img width="1532" height="868" alt="image" src="https://github.com/user-attachments/assets/404170c0-cef7-4355-94f5-a3ade8875ffe" />


vcsa.temu.finance로 접속하자 vSphere Client가 정상 실행된 모습이다.


### 4-2. vCenter VAMI

<img width="1655" height="512" alt="image" src="https://github.com/user-attachments/assets/0a408fcd-1699-4081-822d-94ef27e0c3b7" />


vCenter 관리용 IP인 172.20.100.10:5480으로 접속하여 확인한 모습이다.


## 5. 트러블슈팅
### 5-1. Thin Disk Mode
#### 오류 내용

- iSCSI 용량 부족 - 약 600GB를 한번에 다 써버린 것으로 나옴
- iSCSI에 할당된 600GB를 vCenter가 모두 차지해버리는 상황 발생
#### 원인

- 설치 당시 Thin Disk Mode가 활성화되지 않음
- 설치 과정에서 600GB에 달하는 수준의 블럭이 미리 프로비저닝되어 공간을 차지

#### 해결 방법
- Thin Disk Mode 활성화하여 재설치 진행
- 이후 설치했던 DVS, vSwitch 오류 수정


## 6. 참고사항
- 현재 설치한 환경은 중첩된 가상화 환경에서의 설치
- 실제 ESXi를 구동하는 물리 서버를 관리하기 위한 vCenter 설치는 다르게 진행해야 할 수 있음
- vCenter를 설치한 뒤 ESXi에 접속할 경우 갑자기 설정이 변경될 수 있다는 경고 출력 - 정상적인 상황.

