# ESXi 클러스터 구성

## 1. 목적

여러 ESXi 호스트를 하나의 클러스터로 묶어 고가용성(HA), 자동 부하 분산(DRS), 공유 스토리지(vSAN), 무중단 유지보수 환경을 구성한다.


## 2. 구성도 / 개념

### 클러스터 구성도

<img width="661" height="558" alt="image" src="https://github.com/user-attachments/assets/780c6db1-f573-4867-8714-1ccaee814c3f" />


### 클러스터 도입 효과

| 기능 | 설명 |
| -------- | ----- |
| HA (High Availability) | 호스트 장애 시 클러스터 내 다른 호스트에서 VM 자동 재시작 |
| DRS (Distributed Resource Scheduler) | 호스트들의 CPU/메모리 자원을 단일 풀로 통합하고 부하를 자동 분산 |
| vSAN | 각 호스트의 로컬 디스크를 묶어 가상 Shared Storage 구성, VM별 RAID 정책 자유롭게 선택 가능 |
| 무중단 유지보수 | 호스트를 유지보수 모드로 전환하면 VM이 자동으로 다른 호스트로 이동되어 서비스 중단 없이 작업 가능 |

> **클러스터의 핵심은 vMotion — 위 기능들 모두 vMotion을 기반으로 동작한다.**



## 3. 사전 준비사항, 설치, 설정 절차

### 3-1. 사전 준비사항

| 항목 | 내용 |
| -------- | ----- |
| vCenter | 클러스터 관리를 위한 vCenter가 배포되어 있어야 함 |
| ESXi 호스트 | 클러스터에 추가할 호스트가 vCenter에 등록되어 있어야 함 |
| 공유 스토리지 | HA/DRS 활용을 위해 호스트 간 공유 스토리지 구성 권장 (NFS, iSCSI, vSAN 등) |
| 네트워크 | vMotion 전용 네트워크(VMkernel 포트) 사전 구성 필요 | 
| DNS / 시간 동기화 | 모든 호스트의 NTP 및 DNS 설정이 일치해야 함 |

---

### 3-2. 데이터센터에 클러스터 추가
- vSphere Client → 데이터센터 우클릭 → 새 클러스터 선택

<img width="645" height="402" alt="image" src="https://github.com/user-attachments/assets/e5e6b4f6-6e3c-4c62-a86f-641f4179c970" />

---

### 3-3. 클러스터 생성 옵션 설정
- 클러스터 생성 시 아래 기능을 활성화할 수 있다.

| 옵션 | 설명 |
| -------- | ----- |
| DRS | 자동 부하 분산 활성화. 완전 자동 / 부분 자동 / 수동 중 선택 가능 |
| HA | 호스트 장애 시 VM 자동 재시작 활성화 |
| vSAN | 호스트 로컬 디스크를 묶어 공유 스토리지 구성 활성화 |


<img width="1428" height="682" alt="image" src="https://github.com/user-attachments/assets/bc9ed722-4d29-4b4f-9bcb-aa5ad3884308" />


- 클러스터 생성 완료

<img width="1444" height="697" alt="image" src="https://github.com/user-attachments/assets/c752763a-0e94-4e05-987b-a6cdec222426" />

---

### 3-4. 클러스터에 ESXi 호스트 추가
- 방법1 ㅡ 호스트 추가 메뉴 사용

클러스터 우클릭 → 호스트 추가 → 호스트 IP/도메인 입력 → 자격 증명 입력 → 완료

<img width="1065" height="620" alt="image" src="https://github.com/user-attachments/assets/1273d0ef-ecd8-4169-9cda-8be685cf3a47" />

<img width="1437" height="682" alt="image" src="https://github.com/user-attachments/assets/a3714b7a-bb9d-4569-9e95-713197dff50a" />


- 방법2 ㅡ 드래그 앱 드롭

인벤토리에서 추가할 ESXi 호스트를 클러스터 안으로 드래그 앤 드롭

<img width="1310" height="562" alt="image" src="https://github.com/user-attachments/assets/873844fd-4df4-4aff-ae72-689c59b53e5b" />



## 4. 확인 / 검증 방법

### 4-1. 클러스터 구성 확인

- vSphere Client → 클러스터 선택 → 요약 탭에서 호스트 수, 총 CPU/메모리 자원 확인
- 구성 탭에서 DRS / HA / vSAN 활성화 여부 확인

---

### 4-2. vMotion 동작 확인
```bash
# ESXi 호스트 Shell 또는 vSphere Client에서 수동 vMotion 테스트
# vSphere Client → VM 우클릭 → 마이그레이션 → 호스트 변경
```

- 대상 호스트를 선택 후 마이그레이션 실행
- 서비스 중단 없이 VM이 이동되면 정상

---

### 4-3. HA 동작 확인

- 테스트용 VM을 특정 호스트에서 실행 중인 상태에서 해당 호스트의 네트워크를 의도적으로 차단
- 다른 호스트에서 VM이 자동 재시작되면 HA 정상 동작


## 5. 참고사항

- vMotion이 클러스터의 핵심 — HA, DRS, 유지보수 모드 모두 vMotion을 기반으로 동작하므로 vMotion 네트워크가 안정적으로 구성되어 있어야 함
- DRS 자동화 수준 — 초기 구성 시에는 수동 모드로 설정해 DRS 추천을 검토한 뒤 이상이 없으면 완전 자동으로 전환하는 것을 권장
- HA Admission Control — 호스트 장애 발생 시 VM 재시작에 필요한 여유 자원을 확보하기 위한 정책으로, 클러스터 내 최소 1개 호스트분의 자원을 항상 예약해두는 것이 일반적
- vSAN 구성 시 — 각 호스트에 최소 1개의 캐시용 디스크(SSD)와 1개 이상의 용량 디스크가 필요하며, 최소 3개 이상의 호스트가 권장됨
