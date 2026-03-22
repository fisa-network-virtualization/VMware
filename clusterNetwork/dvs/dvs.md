# DVS Configuration

## 1. 목적

vCenter 환경에서 여러 대의 ESXi 호스트 네트워크를 중앙 집중식으로 관리하기 위해 **DVS(Distributed Virtual Switch, 분산 가상 스위치)** 를 구성한다.

핵심 목적과 기대 효과는 다음과 같다.

- **중앙 집중 관리** : ESXi 호스트의 수가 많아져도 DVS에 포트 그룹(Port Group)을 한 번만 만들면 모든 호스트에 동일한 네트워크 설정이 적용된다.

- **관리 포인트 감소** : 관리해야 할 가상 스위치의 개수가 논리적으로 1개가 된다. 만약 표준 스위치(vSwitch)만 사용한다면, 다음과 같이 ESXi 호스트가 늘어날 때마다 매번 동일한 네트워크 설정을 반복해야 하는 비효율이 발생한다.

  <img width="1124" height="476" alt="image" src="https://github.com/user-attachments/assets/669a8272-252d-4410-b1d4-5f3b5b8ce3df" />


## 2. 구성도 / 개념

### 2-1. DVS 계층 구조

- **DataCenter > Cluster > ESXi Host** 의 계층 구조를 가진다.

- DVS는 **데이터센터(DataCenter)** 레벨에서 생성된다. 따라서 1개의 DVS로 여러 클러스터를 아우르거나, 클러스터에 속하지 않은 호스트들까지도 한 번에 묶어서 사용할 수 있다.

### 2-2. VMkernel (vmk)

- VMkernel 어댑터는 가상 머신(VM)의 IP가 아닌, ESXi 호스트 자체의 통신용 IP를 가지는 인터페이스다. vMotion, Storage(iSCSI), FT 등의 서비스 트래픽을 처리하는 데 사용된다.

## 3. 사전 준비사항, 설치 또는 설정 절차

### 3-1. 외부 ESXi vSwitch 보안 및 Trunk 설정 (중첩 가상화 사전 작업)

현재 인프라는 중첩 가상화(Nested Virtualization) 환경이므로, 내부 클러스터의 DVS 트래픽이 외부(Physical Layer 역할을 하는 ESXi)를 정상적으로 통과할 수 있도록 외부 ESXi의 vSwitch0 설정을 변경해야 한다.

#### 1) Trunk 포트 그룹 구성

<img width="1003" height="439" alt="image" src="https://github.com/user-attachments/assets/36344b41-36ed-4461-917f-2fdf073f68df" />

- 외부 ESXi의 vSwitch0(내부망)에 DVS 업링크 전용 포트 그룹을 추가한다.

- VLAN ID를 **모두(4095)** 로 설정하여 모든 VLAN 트래픽이 통과할 수 있는 Trunk 포트로 구성한다.

#### 2) L2 보안 정책 허용

<img width="1244" height="499" alt="image" src="https://github.com/user-attachments/assets/f05a3c1e-2c3c-4ff9-aa0d-72ef12e8fca8" />

- 생성한 포트 그룹의 보안 설정에서 다음 3가지 정책을 모두 **수락(Accept)** 으로 변경한다.
    - `Promiscuous mode` (비규칙 모드)
    - `MAC address changes` (MAC 주소 변경)
    - `Forged transmits` (위조된 전송)

- **이유:** 중첩 가상화 환경에서는 외부 ESXi의 vSwitch가 내부 VM들이 생성하는 다양한 MAC 주소 트래픽을 차단하지 않고 정상적으로 스위칭해야 하기 때문이다.

### 3-2. DVS 생성 단계

1. **네트워킹 뷰로 이동:** vCenter Server 왼쪽 메뉴에서 Networking(네트워킹) 아이콘을 클릭한다.

<img width="1013" height="504" alt="image" src="https://github.com/user-attachments/assets/881d1ef0-eae7-4d53-8459-a5361785da88" />

2. **데이터 센터 선택:** DVS를 생성할 데이터 센터(DC)를 우클릭하고 Distributed Switch > New Distributed Switch를 선택한다.

3. **이름 및 위치:** DVS의 이름을 입력하고 Next를 클릭한다.

4. **버전 선택:** 환경에 맞는 최신 DVS 버전을 선택한다. (최신 기능 사용을 위해 최상위 버전 권장)

5. **설정 구성:**
- Number of uplinks (업링크 수): 호스트당 사용할 물리적 NIC 수를 설정한다. (일반적으로 이중화를 위해 2개 이상 권장)

- Network I/O Control: 필요시 활성화한다.

- Create a default port group: 체크하면 기본 분산 포트 그룹이 자동 생성된다. (필요 없다면 체크 해제 후 나중에 개별 생성 가능)

6. **완료:** 설정을 검토하고 Finish를 클릭하여 생성을 완료한다.

### 3-3. 분산 포트 그룹(Distributed Port Group) 추가 및 구성


<img width="1060" height="575" alt="image" src="https://github.com/user-attachments/assets/93cf2467-88ae-4a02-9aab-f98a087fbe0a" />

1. 생성된 DVS를 우클릭하여 용도별로 필요한 포트 그룹을 추가한다.

2. 각 포트 그룹 특성에 맞게 VLAN ID를 할당한다. (예: Storage VLAN 연결용)

<img width="452" height="315" alt="image" src="https://github.com/user-attachments/assets/89ecbcbc-25b4-4367-a17a-43b758f2ea88" />

3. **업링크 이중화 설정:** DVS의 각 포트 그룹마다 할당된 2개의 업링크를 환경에 맞춰 Active-Active 또는 Active-Standby 형태로 설정한다.

### 3-4. ESXi 호스트 추가 및 물리 어댑터 할당

<img width="948" height="504" alt="image" src="https://github.com/user-attachments/assets/3ee09d77-f25d-4db8-aab4-00ccbfbe5a76" />

1. 만들어진 DVS를 우클릭하고 Add and Manage Hosts를 선택하여 클러스터 내의 ESXi 호스트들을 추가한다.

<img width="1434" height="687" alt="image" src="https://github.com/user-attachments/assets/184d03ee-b4cf-4168-8850-31a795e88ea4" />

<img width="1440" height="686" alt="image" src="https://github.com/user-attachments/assets/fbb628fd-3d81-4dee-955f-42863a07f4d8" />


2. 추가한 ESXi 호스트들의 각 물리 어댑터(vmnic)를 DVS의 업링크(Uplink)에 매핑 및 할당한다.

>⚠️ 주의: vCenter가 클러스터 내부에 있는 경우 통신 단절을 막기 위해 5-1 트러블슈팅의 절차를 반드시 참고할 것.

### 3-5. VMkernel 어댑터 및 VM 마이그레이션

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e06047b1-a060-45b2-a2d0-f0341f18993f" />


기존 표준 스위치(vSwitch0)에 있던 VMkernel 포트(vmk)와 가상 머신(VM)들을 DVS의 각 목적에 맞는 포트 그룹으로 마이그레이션한다.

- vMotion-PG ⇒ vMotion 전용 vmk 할당 및 서비스 체크
- Storage-PG ⇒ Shared Storage(iSCSI) 연결용 vmk 할당
- FT-PG ⇒ FT 동기화 통신용 vmk 할당

*VMkernel 추가 방법:* 각 호스트를 선택하고 VMkernel 어댑터를 추가한 뒤, DVS 포트 그룹을 선택하고 해당 목적에 맞는 서비스(예: vMotion)를 체크한 후 IP를 할당한다.

## 4. 확인 / 검증 방법

- **네트워크 토폴로지 확인:** vCenter 네트워킹 뷰에서 해당 DVS의 토폴로지 맵을 열어 모든 ESXi 호스트와 VM, VMkernel이 올바른 포트 그룹과 업링크에 매핑되었는지 시각적으로 확인한다.

- **통신 테스트:** ESXi 호스트의 SSH에서 vmkping 명령어를 활용하여 Storage IP(TrueNAS)나 다른 호스트의 vMotion IP 등과 정상적으로 통신이 되는지 확인한다.

## 5. 트러블슈팅

### 5-1. 네트워크 마이그레이션 중 설정 Rollback 발생

#### 오류 내용

기존 표준 스위치(vSwitch0)에서 새롭게 생성한 DVS로 호스트의 물리 어댑터, 커널 포트, VM들을 마이그레이션 하는 도중, vCenter 및 ESXi 호스트와의 연결이 잠시 끊어지면서 **마이그레이션했던 모든 네트워크 설정이 이전 상태로 되돌아가는(Rollback) 문제**가 발생함.

#### 원인

- 관리 주체인 vCenter VM 자체가 설정 대상인 클러스터 내부(ESXi 위)에 존재하는 상태임.

- ESXi에는 물리 어댑터가 2개 있고, `vSwitch0`이 그중 1개를 통해 외부 스위치와 통신 중이었음.

- 마이그레이션 과정에서 통신 중이던 물리 어댑터 2개를 한꺼번에 DVS 업링크로 변경해버리면서, 그 순간 vCenter와 ESXi 간의 통신이 단절됨.

- vCenter는 네트워크 변경 작업 중 호스트와의 연결이 소실되면, 호스트 고립을 막기 위해 안전 장치로 설정을 원상복구(Rollback)하도록 설계되어 있음.

#### 해결 방법

통신 단절이 발생하지 않도록 물리 어댑터를 순차적으로 하나씩 DVS로 넘기는 방식을 사용해야 한다.

1. ESXi의 `vSwitch0`가 현재 연결된 물리 어댑터(예: vmnic0)는 그대로 놔둔다.

2. 남은 1개의 여분 물리 어댑터(예: vmnic1)만 DVS 업링크로 할당한다.

3. 이 상태가 되면 서버가 외부 스위치와 통신할 수 있는 통로가 2군데(vSwitch0, DVS)가 된다.

4. 이 환경을 유지한 상태로 커널 포트(vmk)와 통신 주체인 **vCenter VM 자체를 DVS의 포트 그룹으로 마이그레이션** 한다. (연결이 끊기지 않고 자연스럽게 DVS 망으로 이동됨)

5. 마이그레이션이 완료되면 기존 ESXi에서 쓸모없어진 `vSwitch0`를 제거한다.

6. `vSwitch0`에서 해제된 나머지 물리 어댑터 1개(vmnic0)도 마저 DVS 업링크에 할당하여 이중화를 완성한다.

## 6. 참고사항

- 데이터스토어(iSCSI) 연결 시, 트래픽 분리와 성능 보장을 위해 일반 통신망과 분리된 Storage 전용 VLAN을 DVS 포트 그룹에 설정해야 한다.

- 모든 구성은 중첩 가상화 환경 기준이므로, 물리적인 스위치 장비에 연결할 때는 물리 스위치의 포트에도 Trunk 및 LACP 설정 등이 동일하게 맞춰져야 한다.
