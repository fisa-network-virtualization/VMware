# VMware 실습
- 기간 : 2026.03.05 ~ 2026.03.13

- 개요 :
본 프로젝트는 단일 물리 서버 내에 중첩 가상화(Nested Virtualization) 기술을 활용하여, 실제 데이터센터와 동일한 수준의 고가용성(HA) 및 철저한 망 분리 보안이 적용된 프라이빗 클라우드(VPC) 인프라를 구축하는 것을 목적으로 합니다. 단순한 가상 머신(VM) 생성을 넘어, 네트워크(L2/L3), 공유 스토리지(iSCSI), 방화벽 보안 정책, 인프라 공통 서비스(NTP, DNS)까지 직접 설계하고 제어함으로써 가상 인프라 아키텍처의 완벽한 통제력을 확보했습니다.

<br />

## 👩🏻‍💻 팀원 소개

| <img src="https://avatars.githubusercontent.com/u/50224952?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/180767288?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/127292182?v=4" width="150" height="150"/> |  <img src="https://avatars.githubusercontent.com/u/118096607?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/113974663?v=4" width="150" height="150"/> | <img src="" width="150" height="150"/> |
| :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |  
|                 이명진<br/>[@septeratz](https://github.com/septeratz)                 |                       조민우<br/>[@minwoo-00](https://github.com/minwoo-00)                       |                배기영<br/>[@bbky323](https://github.com/bbky323)                |                서가영<br/>[@caminobelllo](https://github.com/caminobelllo)                |                양규리<br/>[@ygreee0320](https://github.com/ygreee0320)                |                이동욱<br/>[@cuterrabbit](https://github.com/cuterrabbit)                |   


<br />
<hr />

## 🌐 아키텍처 
![구조도_page-0001](https://github.com/user-attachments/assets/f864ac69-72f7-459e-9ae7-702c9775d3c6)

<br />
<hr />

## 📋 목차
### 환경 구축 기초
- [TrueNAS 설치](./env/trueNAS/trueNAS.md)
- [pfSense 설치 및 방화벽 설정](./env/pfSense/pfSense.md)
- [NTP 설정](./env/ntp/ntp.md)
- [DNS 설정](./env/dns/dns.md)
- [vCenter 설치](./env/vCenter/vCenter.md)

### 클러스터 및 네트워크 구성
- [클러스터 생성](./clusterNetwork/cluster/cluster.md)
- [DVS 구성](./clusterNetwork/dvs/dvs.md)
- [Jumbo Frame 설정](./clusterNetwork/jumboFrame/jumboFrame.md)

### 서비스 연동 및 기능 확장
- [NFS 기반 Web Server 동기화](./service/nfs/nfs.md)
- [로드밸런서 구성](./service/nlb/nlb.md)

### 운영 실습 / 미션
- [계정 생성, Tag, Template](./mission/misison1/mission1.md)
- [vMotion, FT 테스트](./mission/mission2/mission2.md)
- [vCenter 백업](./mission/mission3/mission3.md)

### 트러블슈팅
- [문서별 오류 사례 및 해결](./troubleShooting/troubleShooting1/trouble1.md)
