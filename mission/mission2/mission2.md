# Mission 2 - vMotion, FT Testing

## 1. 목적

본 문서(Mission 2)는 구성된 vSphere 클러스터 환경에서 가상 머신(VM)의 서비스 무중단 마이그레이션과 장애 조치 기능을 검증하는 것을 목적으로 한다.\
이를 위해 **vMotion**, **DRS(Distributed Resource Scheduler)**, 그리고 **FT(Fault Tolerance)** 기능을 테스트하고 그 결과를 확인하였음.

## 2. vMotion 및 DRS 테스트

vMotion을 통해 서비스 중단 없이 가상 머신을 다른 ESXi 호스트로 실시간 마이그레이션하는 테스트를 진행함.

### 2-1. 테스트 시나리오

<img width="1370" height="392" alt="image" src="https://github.com/user-attachments/assets/0a768d73-4686-46a7-9fdd-c57efcb2d3ce" />

* **대상 VM 마이그레이션**: 102 호스트에서 실행 중이던 Linux VM(Rocky Linux)을 103 호스트로 vMotion을 사용하여 이동.  
* **DRS 자동 개입 확인**: 클러스터 리소스를 최적화하기 위해, 시스템(DRS)이 103 호스트에 있던 Windows VM을 자동으로 102 호스트로 마이그레이션하는 것을 확인.

### 2-2. 통신 상태 확인 (Ping 테스트)

마이그레이션 중 네트워크 단절 여부를 확인하기 위해, 별도의 Ubuntu VM에서 이동 중인 Rocky Linux VM으로 Ping을 지속적으로 보내는 상태에서 vMotion을 실행.

* **결과**: vMotion이 진행되고 호스트가 변경되는 와중에도 Ping 통신이 끊기지 않고 정상적으로 유지되었음.


> 💡 **기술적 배경: vMotion 중 Ping이 끊기지 않는 이유**
> 
> 1. **메모리 사전 복사 (Pre-copy)**: VM이 소스 호스트에서 정상 실행되는 상태에서 메모리 페이지를 대상 호스트로 복사하므로 네트워크 서비스에 영향이 없습니다. 
> 2. **반복 복사 (Iterative copy)**: 1단계 이후 새롭게 변경된(dirty) 메모리 페이지만 반복 전송합니다. 변경량이 충분히 줄어들 때까지 이 과정을 반복합니다.  
> 3. **순간 전환 (Switchover)**: 변경 페이지가 거의 없어지면, VM을 아주 짧은 순간(보통 1초 미만) 멈추고(stun) 나머지 상태를 전송한 뒤 대상 호스트에서 즉시 재개합니다. 이 전환 시간이 Ping의 기본 타임아웃(1초)보다 짧기 때문에 패킷 손실이 거의 발생하지 않습니다.  
> 4. **네트워크 전환**: 전환 직후 RARP 패킷을 브로드캐스트하여 물리 스위치의 MAC 테이블을 업데이트하므로 트래픽이 즉시 새로운 호스트로 전달됩니다.

## 3. Fault Tolerance (FT) 테스트

Fault Tolerance(무중단 보호)는 VM의 실시간 복제본(보조 VM)을 다른 호스트에서 동시에 실행시켜, 기본 호스트 장애 시 즉시 보조 VM이 기본 VM으로 승격되는 고가용성 기능이다.

### 3-1. FT 설정 및 구성 상태

<img width="2048" height="784" alt="image" src="https://github.com/user-attachments/assets/d740fc44-c6c3-4949-88ff-0365c13388d0" />

* **사전 조건**: FT 기능을 사용하기 위해서는 클러스터의 **vSphere HA** 기능이 반드시 활성화되어 있어야 함.  
* **대상 VM**: Windows VM
* **배치 상태**:

  <img width="1068" height="532" alt="image" src="https://github.com/user-attachments/assets/38d3e5f4-93ab-4257-8980-088b0f05c5d6" />
  
  - **기본(Primary) VM:** 102 호스트에 위치
  
  <img width="1195" height="444" alt="image" src="https://github.com/user-attachments/assets/ce5f6e43-7d34-4544-a4c8-270e341b1f39" />
  
  - **보조(Secondary) VM:** 104 호스트에 위치

### 3-2. FT 페일오버(Failover) 테스트 및 결과

<img width="1007" height="650" alt="image" src="https://github.com/user-attachments/assets/a419e1ce-b170-4690-802c-e30296c9b1a3" />

고의로 장애 상황을 유도하거나 수동으로 페일오버를 트리거하여 전환 테스트를 진행 가능.\
이 상황에선 페일오버 테스트 옵션으로 진행.

<img width="1828" height="772" alt="image" src="https://github.com/user-attachments/assets/124bc03f-d654-4ba0-bc7d-dbb173518a6e" />

<img width="718" height="455" alt="image" src="https://github.com/user-attachments/assets/5787199e-265d-46c8-904c-98ad1ecbfe6d" />


* **현상**: 평소 10ms 대를 유지하던 통신이 페일오버 발생 순간 373ms로 일시적인 지연(튐 현상)이 발생했으나, 직후 1ms 이하로 즉시 감소했음.  
* **원인**: 보조 VM이 위치한 대상 호스트와의 물리적(또는 논리적 네트워크) 거리가 가까워져 전환 이후 오히려 통신 시간이 감소한 것으로 분석됨.

<img width="1188" height="593" alt="image" src="https://github.com/user-attachments/assets/5bed8ae1-c124-443c-92f0-914001a6ad8c" />


* **상태 변화**:  
  * 104 호스트에 대기 중이던 보조 VM이 기본(Primary) VM으로 즉시 승격되어 서비스를 무중단으로 재개했음.  
  * 원래 기본 VM이 있던 102 호스트에서는 새로운 보조(Secondary) VM이 생성되어 실행 상태로 변경됨을 확인함.

| 항목 | 측정 결과 | 비고 |
| :---- | :---- | :---- |
| **페일오버 전 Ping** | \~10ms | 평상시 응답 속도 |
| **페일오버 순간 Ping** | **373ms** (최대 지연) | 전환이 이루어지는 찰나의 순간 |
| **페일오버 후 Ping** | \~1ms 이하 | 새로운 호스트에서 서비스 재개 |
| **패킷 손실** | **0건** | 네트워크 단절 없음 |
| **최종 판정** | ✅ **성공** | 373ms 지연은 TCP 연결이 끊기지 않는 허용 수준임 |

## 4. 클러스터 Affinity 설정 및 DRS 검증

<img width="1144" height="604" alt="image" src="https://github.com/user-attachments/assets/1610a5dd-55dc-4ba3-8865-7c38b6cd3069" />

DRS(분산 리소스 스케줄러)가 클러스터의 리소스를 어떻게 분배할지(How) 결정한다면, Affinity Rule(선호도 규칙)은 특정 VM이 어느 호스트에 배치되어야 할지(Where)를 명시적으로 지정하는 것이다.

<img width="1395" height="692" alt="image" src="https://github.com/user-attachments/assets/33d8dbff-50d8-481d-ab65-13c410d94cbe" />


### 4-1. DRS 수동 모드 vs Affinity Rule 비교

| 항목 | DRS 수동 모드 (Manual Mode) | Affinity Rule (선호도 규칙) |
| :---- | :---- | :---- |
| **목적** | 자동 마이그레이션(이동)을 **막는 것** | VM의 배치 위치를 명시적으로 **지정하는 것** |
| **설정 대상** | 클러스터 전체 | 특정 VM 또는 VM 그룹 단위 |
| **동작 방식** | 시스템에 의한 자동 마이그레이션 비활성화 | 특정 호스트(또는 호스트 그룹)에 배치를 강제하거나 권장 |
| **사용 시점** | 전체 클러스터의 자동화를 일시적으로 중단할 때 | 라이선스, 보안, 특정 하드웨어 성능 등 배치 제약이 있을 때 |

### 4-2. DRS 수동 상태에서의 권장 사항 적용 테스트

<img width="1499" height="635" alt="image" src="https://github.com/user-attachments/assets/35533c2a-b3c8-49e5-8737-c11767eed89b" />
<img width="1880" height="580" alt="image" src="https://github.com/user-attachments/assets/4473a8e3-fe2a-4877-94fd-1462ee256293" />


DRS를 수동 모드로 둔 상태에서, 시스템이 제시하는 리소스 최적화 '권장 사항'을 수동으로 실행하여 그 결과를 확인하였음.

| 항목 | 실행 전 (권장 사항 대기) | 실행 후 (권장 사항 적용 완료) |
| :---- | :---- | :---- |
| **대상 VM** | linux-rocky-02 | linux-rocky-02 |
| **DRS 점수** | **2%** | **89%** |
| **상태 판정** | ⚠️ 리소스 불균형 심각 | ✅ 최적 배치 및 리소스 안정화 |

<img width="1516" height="674" alt="image" src="https://github.com/user-attachments/assets/5b7f2b80-3d64-4786-b198-ccc5b7ef7b17" />

VM DRS 점수가 변화한 것을 확인할 수 있음.
