# Jumbo Frame 설정

## 1. 목적

이더넷 프레임의 기본 MTU(1500 bytes)를 9000 bytes까지 확장하여 Storage / vMotion / FT 등 대용량 트래픽이 지속적으로 발생하는 네트워크 구간의 전송 효율과 성능을 향상시킨다.

### 적용 대상 포트그룹

| 포트그룹 | 이유 |
| -------- | ----- |
| Storage-PG | GB~TB단위의 디스크 블록 데이터를 지속적으로 읽고 씀 |
| vMotion-PG | vMotion 시 수십 GB의 VM 메모리를 한 번에 복사해야 함 |
| FT-PG | Primary <-> Secondary VM 간 상태를 실시간으로 동기화함 |

### 기대 효과

- CPU 부하 감소 — 패킷을 잘게 쪼개고 재조립하는 횟수가 줄어 ESXi 호스트의 CPU 사용량이 감소
- 전송 효율 향상 — 헤더 오버헤드 비율이 낮아지고 실제 데이터 비율이 높아져 처리량 증가

## 2. 구성도 / 개념

### 구성 및 흐름

<img width="1061" height="779" alt="image" src="https://github.com/user-attachments/assets/f61d49f6-acdd-43b0-899c-cc3dc00629fc" />

### 핵심 원칙

End-to-End 설정 — 
Jumbo Frame은 경로상 모든 구간의 MTU가 9000으로 통일되어 있어야 정상 동작한다.
중간에 MTU 1500 구간이 존재하면 아래 문제가 발생한다.
- Fragmentation 발생 → 성능 저하
- DF(Don't Fragment) 비트가 설정된 패킷 → 패킷 Drop



## 3. 사전 준비사항, 설치, 설정 절차

현재 구성에서 MTU 9000 설정이 필요한 지점은 총 4곳이다.

### 3-1. vSwitch0(내부망) MTU 설정

- 설정 방법: vSphere Client → 해당 호스트 → 구성 → 네트워킹 → vSwitch0 편집 → MTU를 9000으로 변경
<img width="1275" height="851" alt="image" src="https://github.com/user-attachments/assets/bc2a3443-28f9-433f-802d-3dffcc24de39" />

---

### 3-2. DVS MTU 설정

- 설정 방법: vSphere Client → 네트워킹 → DVS 선택 → 설정 편집 → MTU를 9000으로 변경
<img width="1064" height="670" alt="image" src="https://github.com/user-attachments/assets/cd9f0eb8-495f-40b3-bbd3-4752190b5803" />

---

### 3-3. DVS 포트그룹의 커널 포트 MTU 설정

- 설정 방법: vSphere Client → 네트워킹 → 각 포트그룹(Storage-PG / vMotion-PG / FT-PG) 선택 → VMkernel 어댑터 편집 → MTU를 9000으로 변경
<img width="1091" height="691" alt="image" src="https://github.com/user-attachments/assets/d84f3877-3776-412d-8e2f-9b57dcc46d3c" />

---

### 3-4. TrueNAS MTU 설정

- 설정 방법: TrueNAS 관리 콘솔 → Network → Interfaces → 해당 인터페이스 편집 → MTU를 9000으로 변경
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/dce4ef93-7d34-4131-b20b-4857b6409ca1" />

---

## 4. 확인 / 검증 방법

### 4-1. ssh로 ESXi 호스트에 접속 후 MTU 확인
```bash
esxcfg-vmknic -l
```
<img width="1483" height="762" alt="image" src="https://github.com/user-attachments/assets/16488bb1-89db-40c3-b0ab-08d68056b951" />

- MTU 가 9000으로 설정되어 있는 것을 확인 가능

---

### 4-2. Ping 테스트
```bash
vmkping -I vmk4 -d -s 8972 10.40.130.10
```

- -d 옵션: “Do not Fragment” 비트를 설정, 이 설정 때문에 8900바이트의 핑이 그대로 넘어감. 점보 프레임이 설정되어 있지 않은 경우 실패하고 그냥 패킷을 드랍

<img width="770" height="229" alt="image" src="https://github.com/user-attachments/assets/b261cccc-9c79-4d78-a26d-1afda88cf2a1" />

- 8980 bytes의 패킷이 응답으로 돌아온 것을 확인 가능

## 5. 참고사항
- ICMP 핑 헤더는 8 bytes를 차지하므로, MTU 9000 환경에서 최대 페이로드 테스트 시 -s 8972를 사용한다 (9000 - 20(IP 헤더) - 8(ICMP 헤더) = 8972)
- Jumbo Frame은 End-to-End 설정이 필수이며, 경로 상 단 하나의 구간이라도 MTU 1500이면 효과가 없거나 오히려 장애가 발생할 수 있다
- 만약 트래픽이 서버 외부로 나간다면 물리 스위치에서도 Jumbo Frame(MTU 9000) 설정이 필요함.


