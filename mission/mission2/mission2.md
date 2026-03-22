# vMotion, FT 테스트

## vMotion 테스트

vMotion을 통해 서비스 중단 없이 가상 머신(VM)을 다른 호스트로 마이그레이션한다.

### 테스트 시나리오

- `102` 호스트에서 실행 중이던 Linux VM을 `103` 호스트로 vMotion 실행
- **DRS 자동 개입**: 시스템이 클러스터 리소스를 최적화하기 위해, `103`에 있던 Windows VM을 자동으로 `102` 호스트로 vMotion 실행
<img width="1370" height="392" alt="image" src="https://github.com/user-attachments/assets/3a1b9830-1bd2-421c-b7b4-8bfc977be761" />


### 통신 상태 확인

Ubuntu VM에서 해당 Rocky Linux VM으로 Ping을 지속적으로 보내며 vMotion을 실행하여 네트워크 단절 여부를 확인한다.

**Ping 테스트 결과**

- vMotion 실행 전후로 패킷 손실은 0건으로, 서비스 연속성이 유지된다.
- vMotion 진행 중에도 Ping 응답 시간에 유의미한 변화가 없으며, 네트워크 단절 없이 마이그레이션이 완료된다.
- 이를 통해 vMotion이 실시간 서비스 환경에서도 무중단으로 동작함을 확인한다.

### vMotion 중 Ping이 끊기지 않는 이유

1. **메모리 사전 복사 (Pre-copy)**: VM이 실행되는 상태에서 메모리 페이지를 대상 호스트로 복사한다. 소스 호스트에서 정상 동작 중이므로 네트워크 영향이 없다.
2. **반복 복사 (Iterative copy)**: 1단계 이후 변경된(dirty) 메모리 페이지만 반복 전송한다. 변경량이 충분히 줄어들 때까지 반복한다.
3. **순간 전환 (Switchover)**: 변경 페이지가 거의 없어지면, VM을 아주 짧은 순간(보통 1초 미만) 멈추고(stun) 나머지 상태를 전송한 후 대상 호스트에서 즉시 재개한다. 이 시간이 Ping 기본 타임아웃(1초)보다 짧아 패킷 손실이 거의 없다.
4. **네트워크 전환**: 전환 직후 RARP 패킷을 브로드캐스트하여 물리 스위치의 MAC 테이블을 업데이트한다. 트래픽이 즉시 새 호스트로 전달된다.

---

## FT(Fault Tolerance) 설정 및 테스트

Fault Tolerance는 VM의 실시간 복제본(보조 VM)을 다른 호스트에서 동시에 실행하여, 호스트 장애 시 즉시 보조 VM이 기본 VM으로 승격되는 무중단 보호 기능이다.

> 💡 FT 사용을 위해서는 클러스터의 vSphere HA 기능 활성화가 필요하다.
<img width="2559" height="979" alt="image (1)" src="https://github.com/user-attachments/assets/5210a25d-fa01-4511-8846-d2b6f8354213" />


### FT 구성 상태

- Windows VM을 대상으로 FT를 설정한다.
<img width="867" height="526" alt="image (2)" src="https://github.com/user-attachments/assets/5ee3c4e3-610e-4c8d-bf56-d41f79f8fc3e" />

- 기본(Primary) VM: `102` 호스트에 위치
- 보조(Secondary) VM: `104` 호스트에 위치
<img width="1828" height="772" alt="image (3)" src="https://github.com/user-attachments/assets/a9e133a6-d7a9-45eb-902e-51257ea7f491" />
<img width="1068" height="532" alt="image (4)" src="https://github.com/user-attachments/assets/75d142c4-81ad-42ad-8ce0-db4a33f4ab09" />
<img width="1195" height="444" alt="image (5)" src="https://github.com/user-attachments/assets/9ef262bf-0e97-402d-b556-8284019de140" />




### FT 페일오버 테스트
<img width="1007" height="650" alt="image (6)" src="https://github.com/user-attachments/assets/17fc32ad-90d7-4e76-a3ff-a35aadd5b1b0" />


페일오버 발생 시 통신 상태를 확인한다.
<img width="719" height="460" alt="image (7)" src="https://github.com/user-attachments/assets/024b1eb0-324b-4b75-af9c-cc93ed41a5af" />


- 평소 ~10ms를 유지하던 통신이 페일오버 발생 순간 373ms로 일시적 튐 현상이 발생하나, 직후 ~1ms 이하로 감소한다.
  → 대상 호스트와의 물리적 거리가 가까워져 통신 시간이 감소한다.
- `104` 호스트에 있던 Windows VM(보조)이 기본 VM으로 즉시 승격되어 서비스를 재개한다.
- 원래 VM(기본)이 있던 `102` 호스트에서 VM(보조)이 실행 중으로 변경된다.
<img width="1173" height="486" alt="image (8)" src="https://github.com/user-attachments/assets/79b8c4b4-4762-4ddf-8116-34ce0afcbb0d" />
<img width="1188" height="593" alt="image (9)" src="https://github.com/user-attachments/assets/7bb230de-eeca-4392-9b1f-82304ccfb3ce" />



| 항목 | 결과 |
|------|------|
| 페일오버 전 Ping | ~10ms |
| 페일오버 순간 Ping | 373ms (최대) |
| 페일오버 후 Ping | ~1ms |
| 패킷 손실 | 0건 |
| 판정 | ✅ 성공 — 373ms는 TCP 연결이 끊기지 않는 수준 |

---

## 클러스터 Affinity 설정

특정 VM을 특정 호스트(그룹)에 배치하도록 강제하거나 권장하는 규칙이다. DRS가 "어떻게 분산할지"를 결정하고, Affinity Rule은 "어디에 배치할지"를 지정한다.

| 항목 | DRS 수동 모드 | Affinity Rule |
|------|---------------|---------------|
| 목적 | 자동 이동을 **막는** 것 | 배치 위치를 **지정하는** 것 |
| 설정 대상 | 클러스터 전체 | VM 또는 VM 그룹 단위 |
| 동작 방식 | 자동 마이그레이션 비활성화 | 특정 호스트에 배치 강제/권장 |
| 사용 시점 | 전체 자동화를 일시적으로 중단할 때 | 라이선스, 보안, 성능 등 배치 제약이 있을 때 |
<img width="1350" height="705" alt="image" src="https://github.com/user-attachments/assets/1b6d3609-8e46-48b4-824c-f5b48a8df7c5" />



DRS 수동 모드에서 권장 사항을 실행한 결과는 다음과 같다.
<img width="1499" height="635" alt="image" src="https://github.com/user-attachments/assets/61d38f68-5a76-4ef2-b69e-7aef6ac0efd0" />
<img width="1516" height="674" alt="image (1)" src="https://github.com/user-attachments/assets/f93598b4-e59c-4e2f-9068-4408a1bec6a1" />




### 권장 사항 실행 결과

| 항목 | 실행 전 | 실행 후 |
|------|---------|---------|
| 대상 VM | `linux-rocky-02` | `linux-rocky-02` |
| DRS 점수 | 2% | 89% |
| 판정 | ⚠️ 리소스 불균형 | ✅ 최적 배치 |
