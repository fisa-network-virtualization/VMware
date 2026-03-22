# pfSense 방화벽 설정

## 1. 목적

- 가상화 환경(VMware vSphere/ESXi) 내부 인프라에서 외부 인터넷 및 내부 네트워크 간 트래픽을 통제하는 방화벽 구성
- 외부에서 내부 웹 서버(프라이빗 IP)에 접근할 수 있도록 NAT 포트 포워딩 설정
- 네트워크 대역별(LAN, DMZ, DB 전용망)로 격리된 보안 정책 적용

<br/>

## 2. 구성도 / 개념

### 인터페이스 구성

| 인터페이스 | 어댑터 | 연결 대상 | IP | 역할 |
|---|---|---|---|---|
| WAN | vmx1 | vSwitch (외부망) | DHCP | 외부 인터넷 통로 |
| LAN | vmx2 | Managed PG (VLAN 100) | 172.20.100.254 | 내부 기본 네트워크 |
| OPT1 | vmx3 | Inter PG (VLAN 110) | 10.40.110.1 | 웹 서버 DMZ 네트워크 |
| OPT2 | vmx0 | VM Network (VLAN 140) | 10.40.140.1 | DB 전용 Private 네트워크 |

### 트래픽 흐름 (내부 → 외부)

```
VM NIC → DVS → ESXi1 NIC → vSwitch0(내부망) → pfSense → vSwitch1(외부망) → 인터넷
```

### 트래픽 흐름 (외부 → 내부, NAT 포트포워딩)

```
인터넷 → vSwitch1(외부망) → pfSense(포트포워딩) → vSwitch0(내부망) → DVS → VM NIC
```

- 원칙적으로 외부 → 내부는 차단이나, 웹 서버 접근을 위해 NAT 포트포워딩으로 예외 허용

<br/>

## 3. 사전 준비사항, 설치 또는 설정 절차

### 3-1. pfSense 선택 이유

- 직관적인 웹 UI
- 방대한 참고문서

### 3-2. 설치

1. netgate 공식 사이트에서 ISO 파일 다운로드 (2026.03 기준 버전 2.8)
2. 가상머신 생성
3. 네트워크 어댑터 추가
   - 인프라 설계에 따라 방화벽은 Managed PG, Inter PG(외부망 스위치용), Inter PG(내부망 스위치용), VM Network과 연결

### 3-3. 웹 GUI 접속

- pfSense 웹 GUI 최초 접속 시 방화벽이 차단할 수 있으므로 콘솔에서 아래 명령어로 일시 비활성화 후 접속

    ```bash
    pfctl -d
    ```

- 접속 주소: `172.20.100.254` (Managed PG에 할당한 LAN 게이트웨이 IP)

### 3-4. NAT 포트포워딩 설정

- 목적: 외부(WAN)에서 들어오는 HTTP(80), HTTPS(443) 트래픽을 OPT1 대역의 웹 서버(10.40.110.30)로 전달
- WAN → 10.40.110.30:80/443 포트포워딩 규칙 생성

### 3-5. 방화벽 규칙 설정

**WAN 규칙**

- NAT 규칙 생성 시 함께 만들어진 방화벽 허용 규칙 (linked rule)
- 외부에서 웹 서버의 80, 443 포트로의 접근을 방화벽 단에서 허용

**LAN 규칙**

- Anti-Lockout Rule: 관리자가 pfSense 웹 GUI에 접속하기 위한 규칙
- LAN to OPT1: LAN 대역 → OPT1 대역 통신 허용

**OPT1 규칙**

- nfs-storage: OPT1 대역 → LAN 대역 172.20.100.88(NFS 스토리지) 전체 통신 허용 (웹 서버에서 NFS 스토리지 읽기/쓰기 목적)
- ICMP(ping 테스트용) 허용
- UDP 123(NTP), 53(DNS) 허용

**OPT2 규칙**

- DB 서버가 위치한 격리 네트워크로 매우 제한적으로 설정
  - 웹 서핑(80, 443), 내부망 Ping 등 기본 통신 전부 차단
  - UDP 123(NTP), 53(DNS) 포트만 허용

<br/>

## 4. 확인 / 검증 방법

### 4-1. 정상 통신 테스트 환경 구성

1. ESXi 2번 Rocky Linux VM (Inter-PG 연결)에서 내부 방화벽 비활성화 후 파이썬 HTTP 서버 실행

    ```bash
    sudo systemctl stop firewalld        # Rocky 내부 방화벽 비활성화 (모든 방화벽 설정을 pfSense에 위임)
    sudo python3 -m http.server 443      # 파이썬 서버 생성
    ```

### 4-2. 외부에서 통신 확인

- 물리 ESXi와 같은 네트워크 대역에 연결된 노트북(192.168.0.6)에서 curl 요청
- Python 서버 로그 및 노트북 터미널에서 **200 OK** 응답 확인

### 4-3. 패킷 통신 확인

- pfSense의 **Packet Capture** 기능으로 패킷 확인
  - ESXi 2번 Rocky Linux VM (Python 서버: 10.40.110.21) ↔ 외부 노트북(192.168.0.6) 간 TCP 통신 정상 확인

<br/>

## 5. 트러블슈팅

### 문제

ESXi 내부 VM(Rocky Linux)에서 pfSense 게이트웨이 IP로 ping이 전송되지 않음

### 원인

포트그룹 보안 설정이 "거부"로 되어 있을 경우, VM이 내보내는 패킷의 MAC 주소를 ESXi가 위조된 패킷으로 판단하여 물리 서버로 전송하지 않음

### 해결

**① DVS 포트그룹 보안 설정 변경**

vCenter에서 DVS 내 해당 VM이 속한 포트그룹(Inter-PG)의 보안 설정을 아래와 같이 변경

- 비규칙 모드(Promiscuous Mode) → **허용**
- MAC 주소 변경(MAC Address Changes) → **허용**
- 위조 전송(Forged Transmits) → **허용**

**② pfSense에서 VLAN 태그 설정 제거**

- pfSense 인터페이스에 불필요한 VLAN 태그가 설정되어 있을 경우 제거 필요 *(상세 내용 정리 중 — 260318)*

---

## 6. 참고사항

- pfSense 공식 문서: https://docs.netgate.com/pfsense/en/latest/
- Rocky Linux 내부 방화벽(firewalld)은 비활성화하여 모든 방화벽 정책을 pfSense에 일원화하는 방식으로 운영