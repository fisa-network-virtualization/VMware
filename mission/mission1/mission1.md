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

# 미션 2 - vCenter VM 태그(Tag) 설정 및 검색

## 1. 목적
vCenter 환경에서 수많은 가상 머신을 용도, 환경, 중요도에 따라 체계적으로 분류하고, 원하는 자원을 신속하게 식별하기 위해 태그 시스템을 구축한다.

<br/>

## 2. 사전 준비사항, 설치 또는 설정 절차

### 2-1. 태그 정책 수립
태그를 만들기 전, 어떤 기준으로 VM을 분류할 지 정의하기 위해 다음과 같은 관리 체계를 수립하였다.

| 카테고리 | 태그 |
| :--- | :--- |
| **환경** | 운영, 개발, 스테이징, DR |
| **용도** | 웹서버, WAS, DB, 파일서버, 백업 |
| **부서** | 인프라팀, 개발팀, 경영지원, 영업 |
| **중요도** | Critical, High, Medium, Low |
| **백업정책** | Daily, Weekly, Monthly, None |

<br/>

### 2-2. 태그 범주(Category) 생성
1. vSphere Client 접속
2. [메뉴] → **[태그 및 사용자 지정 특성(Tags & Custom Attributes)]**
3. [범주(Categories)] 탭 → **[새 범주(New Category)]** 클릭
4. 설정값 입력
<img width="800" alt="category_create" src="https://github.com/user-attachments/assets/de490bf1-c8d9-4c6f-b559-0d1af3be6514" />

<br/>

### 2-3. 태그(Tag) 생성

- ### vSphere Client에서 생성
1. [태그(Tags)] 탭 → **[새 태그(New Tag)]** 클릭
2. 태그 이름 입력, 범주 선택 → [생성]

- ### PowerCLI로 일괄 생성
```bash
# Environment 태그
New-Tag -Name "Dev" -Category "Environment" -Description "개발 환경"
New-Tag -Name "Staging" -Category "Environment" -Description "스테이징 환경"
New-Tag -Name "Prod" -Category "Environment" -Description "운영 환경"

# Service 태그
New-Tag -Name "Web" -Category "Service" -Description "웹 서버"
New-Tag -Name "API" -Category "Service" -Description "API 서버"
New-Tag -Name "DB" -Category "Service" -Description "데이터베이스"
New-Tag -Name "Cache" -Category "Service" -Description "캐시 서버"

# BackupLevel 태그
New-Tag -Name "Critical" -Category "BackupLevel" -Description "매시간 백업"
New-Tag -Name "Standard" -Category "BackupLevel" -Description "일 1회 백업"
New-Tag -Name "None" -Category "BackupLevel" -Description "백업 제외"
```

<br/>

### 2-4. VM에 태그 할당 및 확인
1. VM 선택 → 우클릭 → **[태그 및 사용자 지정 특성] → [태그 할당]**
2. 원하는 태그 선택 → **[할당]**
<img width="800" alt="tag_add" src="https://github.com/user-attachments/assets/eb2cc4bf-3dc1-42cd-a179-6a39daa0e301" />

<br/>

## 3. 확인 / 검증 방법

### 태그 확인
- VM 요약 화면의 태그 부분에서 할당된 태그 확인 가능
<img width="800" alt="tag_conf" src="https://github.com/user-attachments/assets/1729a63d-d44a-49d2-abd1-f8fe3e4cf3a4" />

### vSphere Client에서 태그 기반 검색

1. vSphere Client 상단 **검색(Search)** 바 클릭
2. `tag:` 입력 후 태그 선택 (ex: `tag:Prod`)
3. 해당 태그가 할당된 모든 VM 목록 표시
<img width="800" alt="tag_search" src="https://github.com/user-attachments/assets/d7f49a17-7ac3-47f7-b5df-e1b5554db485" />

<br/>

<hr/>

# 미션 3 - VM Template 생성
vCenter 환경에서 일관된 서버 배포를 위한 **VM Template**을 생성하고, 시스템 복제 시 발생할 수 있는 고유 식별 정보 충돌을 방지하기 위한 OS별 **일반화 과정**을 다룬다.

## 1. 목적
- **배포 효율성 증대**: 동일한 설정을 가진 다수의 가상 머신(VM)을 빠르게 생성하여 인프라 구축 시간을 단축
- **표준화 및 일관성 유지**: 마스터 이미지를 활용하여 모든 서버의 초기 환경(OS 패치, 기본 패키지 등)을 동일하게 유지

## 2. 사전 준비사항, 설치 또는 설정 절차

### 2-1. 일반화 작업
> 템플릿 생성 전, **OS 고유의 식별 정보를 초기화**하는 일반화 작업 필요

### 2-1.A. Windows 일반화
- `C:\Windows\System32\Sysprep`로 이동 → `sysprep.exe` 실행
<img width="700" alt="window_sysprep" src="https://github.com/user-attachments/assets/5dc63862-53d8-4fba-b504-305b02a18baf" />

- [시스템 OOBE(첫 실행 경험) 시작] / [시스템 일반화] / [시스템 종료] 선택

<br/>

### 2-1.B. Linux 일반화
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

