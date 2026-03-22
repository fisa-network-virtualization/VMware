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

# 미션 3 -
