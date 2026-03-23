# 미션 1 - ESXi 계정 생성

## 1. 목적

- 처음 VMware ESXi를 설치하면 기본적으로 root 계정이 생성되며, 보안을 위해 접근 차단 필요 (많은 brute force 공격에 노출될 위험성 존재)
- 새로운 유저를 생성하고 권한을 부여해 사용자 계정을 관리

<br/>

## 2. 사전 준비사항, 설치 또는 설정 절차

### 2-1. 계정 생성
<img width="1320" height="654" alt="image" src="https://github.com/user-attachments/assets/7a4cac5e-6856-45b2-be25-3457cb2415d4" />

- 좌측 메뉴 [관리] → [보안 및 사용자] → [사용자] → [사용자 추가]

### 2-2. 권한 추가
<img width="498" height="637" alt="image" src="https://github.com/user-attachments/assets/02a45cbb-601d-4146-981f-b760f34c87e8" />

- 좌측 호스트 아래 [관리] 우클릭 → [사용 권한] 선택
  - 관리자 / 읽기 전용 설정 가능

### 2-3. root 비활성화

- root 계정에 권한 없음 설정

<br/>

## 3. 확인 / 검증 방법

### 읽기 전용 계정에서 설정 변경 시도
<img width="1217" height="646" alt="image" src="https://github.com/user-attachments/assets/a7ded523-2f59-4b21-b943-84fa168ff166" />

- 읽기 권한만 부여했기 때문에 설정 변경 시도 시 오류 발생

### root 비활성화 (권한 없음 설정) 시
<img width="848" height="370" alt="image" src="https://github.com/user-attachments/assets/5791de63-926e-4497-be7e-899323fcf8fd" />

- 로그인 권한 없음

<br/>

## 4. 참고사항

- https://blog.1nfra.kr/280

<hr/>

# 미션 2 - 

<hr/>

# 미션 3 - VM Template 생성
vCenter 환경에서 일관된 서버 배포를 위한 **VM Template**을 생성하고, 시스템 복제 시 발생할 수 있는 고유 식별 정보 충돌을 방지하기 위한 OS별 **일반화 과정**을 다룬다.

## 1. 목적
- **배포 효율성 증대**: 동일한 설정을 가진 다수의 가상 머신(VM)을 빠르게 생성하여 인프라 구축 시간을 단축
- **표준화 및 일관성 유지**: 마스터 이미지를 활용하여 모든 서버의 초기 환경(OS 패치, 기본 패키지 등)을 동일하게 유지

## 2. 사전 준비사항, 설치 또는 설정 절차

### 2-1. 일반화 작업
> 템플릿 생성 전, **OS 고유의 식별 정보를 초기화**하는 일반화 작업 필요

### 2-1.A Windows 일반화
- `C:\Windows\System32\Sysprep`로 이동 → `sysprep.exe` 실행
<img width="700" alt="window_sysprep" src="https://github.com/user-attachments/assets/5dc63862-53d8-4fba-b504-305b02a18baf" />

- [시스템 OOBE(첫 실행 경험) 시작] / [시스템 일반화] / [시스템 종료] 선택

<br/>

### 2-1.B Linux 일반화
- 터미널에서 아래 명령어를 순차적으로 실행하여 고유 정보를 제거
``` bash
# 1. SSH 호스트 키 및 머신 ID 초기화
rm -f /etc/ssh/ssh_host_*
truncate -s 0 /etc/machine-id

# 2. 네트워크 설정 초기화 (MAC 주소 및 고정 IP 바인딩 제거)
# (Ubuntu/Debian 기준)
rm -f /etc/netplan/*.yaml  # 필요 시 기본 설정만 남기고 제거

# 3. 로그 및 임시 파일, 명령어 기록 삭제
rm -rf /var/log/*
rm -rf /tmp/*
history -c && rm -f ~/.bash_history

# 4. 시스템 종료
shutdown -h now
```

<br/>

### 2-2. 템플릿으로 변환하기

<img width="1000" alt="template1_h" src="https://github.com/user-attachments/assets/559499d3-d38a-4137-b17d-6eade3b1b888" />

- 전원이 꺼진 VM 우클릭 → [템플릿] → [템플릿으로 변환] 클릭
<br/>

<img width="1000" alt="template2_h" src="https://github.com/user-attachments/assets/5c25bb2c-adc1-4127-9448-54170cb2dfdc" />

- VM 아이콘이 일반 가상 머신에서 템플릿 모양으로 변경됨

<br/>

### 2-3. 템플릿에서 새 VM 생성하기
<img width="1911" height="817" alt="template_create1_h" src="https://github.com/user-attachments/assets/236d88cd-50e1-43c6-ae99-89de79f0eddd" />

- 변환된 템플릿 우클릭 → [이 템플릿에서 새 VM 생성..] 선택

<img width="1920" height="821" alt="template_create2" src="https://github.com/user-attachments/assets/12ec0adf-c973-4797-b070-92f1def5082a" />

- 새 VM 이름, 위치, 클러스터, 데이터스토어 등을 지정하여 배포

