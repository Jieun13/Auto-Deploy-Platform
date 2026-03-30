# **🚀 AutoDeploy Platform**

> GitHub Repository URL만 입력하면 자동으로 컨테이너를 빌드하고 배포해주는 서버리스 미니 PaaS

## **서비스 개요**

### **해결하는 문제**

개발자가 새로운 프로젝트를 외부에 배포하려면 다음과 같은 복잡한 작업들을 직접 해야 합니다.

- **인프라 설정의 복잡성**: EC2 인스턴스 생성, 보안 그룹 설정, 네트워크 구성 등 배포와 무관한 인프라 지식이 요구됨
- **Dockerfile 작성 부담**: 프레임워크별로 다른 컨테이너 빌드 설정을 개발자가 직접 작성해야 함
- **배포 파이프라인 부재**: CI/CD 파이프라인 구축 없이는 코드 변경 시마다 수동으로 빌드, 배포를 반복해야 함
- **운영 모니터링 부재**: 배포 이후 서비스 상태(CPU, Memory), 실행 로그를 한눈에 확인하기 어려움
- **에러 원인 파악의 어려움**: 빌드, 배포 실패 시 방대한 로그를 직접 분석해야 하는 진입 장벽

소규모 팀이나 사이드 프로젝트 개발자는 클라우드 인프라 전문 지식 없이도 **코드에만 집중하면서 빠르게 서비스를 런칭**하고 싶어합니다.

### **솔루션**

AutoDeploy Platform은 GitHub Repository URL과 프레임워크 종류만 입력하면 나머지 빌드, 배포 과정을 AWS 네이티브 서비스로 완전 자동화합니다.

| 문제 | 솔루션 |
| --- | --- |
| 인프라 설정 복잡성 | AWS App Runner로 서버 관리 Zero화 |
| Dockerfile 작성 부담 | CodeBuild에서 프레임워크별 Dockerfile 동적 자동 생성 |
| 배포 파이프라인 부재 | CodeBuild → ECR → App Runner 자동 파이프라인 |
| 운영 모니터링 부재 | CloudWatch Metrics 연동 실시간 CPU/Memory 대시보드 |
| 에러 원인 파악 어려움 | Amazon Bedrock이 에러 로그를 한국어로 자동 요약 |

---

## **서비스 시스템 아키텍처**
### **전체 아키텍처 다이어그램**
![architecture](architecture.jpeg)
### **핵심 서비스 구성 요소**

| AWS 서비스 | 역할 |
| --- | --- |
| **S3 + CloudFront** | React 프론트엔드 정적 파일 호스팅 및 글로벌 CDN 배포 |
| **AppSync** | GraphQL API 제공 + DynamoDB 변경을 감지하여 React에 실시간 상태 Push (GraphQL Subscription) |
| **AWS Lambda** | 핵심 비즈니스 로직 처리 (배포 요청, 상태 조회, AI 호출) |
| **AWS CodeBuild** | GitHub 소스 Clone → 프레임워크별 Dockerfile 동적 생성 → Docker 이미지 빌드 |
| **AWS ECR** | CodeBuild가 생성한 Docker 이미지 저장소 |
| **AWS App Runner** | ECR 이미지를 기반으로 컨테이너를 실행하고 공개 URL을 자동 발급 |
| **Amazon DynamoDB** | 프로젝트 메타데이터, 빌드 상태, AI 요약 결과를 저장하는 NoSQL DB |
| **Amazon CloudWatch** | App Runner의 CPU/Memory 메트릭 수집 및 CodeBuild 빌드 로그 저장 |
| **Amazon Bedrock** | 빌드/배포 실패 시 CloudWatch 에러 로그를 Claude 모델로 분석 후 한국어 요약 |


### **주요 워크플로우**

```
[1] 배포 요청
    사용자 → (GitHub URL, Framework 선택) → Frontend → AppSync
    → Lambda: DynamoDB에 status: PENDING 기록 + CodeBuild 실행 트리거

[2] 자동 빌드 (CodeBuild)
    → GitHub Repository Clone
    → 선택된 Framework에 맞는 Dockerfile 동적 생성
      (예: Spring Boot → JDK 이미지 기반 멀티스테이지 빌드 스크립트 자동 작성)
    → Docker Build & AWS ECR에 이미지 Push

[3] 자동 배포 (App Runner)
    → ECR에 새 이미지 Push 감지
    → App Runner가 컨테이너를 실행하고 공개 URL 자동 발급

[4] 상태 동기화 (AppSync)
    → CodeBuild / App Runner의 상태 변경 이벤트 발생
    → App Runner → Lambda (배포 완료 알림/트리거)
    → Lambda가 DynamoDB 업데이트
    → DynamoDB 변경을 AppSync가 감지
    → AppSync가 React에 GraphQL Subscription으로 실시간 Push

[5] 사용자 확인
    → Frontend에서 실시간 상태 확인
    → SUCCESS: 배포 URL 제공 + CPU/Memory 모니터링 차트
    → FAILED: AI가 분석한 에러 원인 한국어 텍스트 표시
```


## **기여자**

| 이름 | GitHub | 담당 영역 |
|------|--------|-----------|
| 김민서 | [galllee](https://github.com/galllee) | 빌드 파이프라인 |
| 백지은 | [Jieun13](https://github.com/Jieun13) | 배포 & 상태 관리 |
| 이승현 | [nanami-tomoe](https://github.com/nanami-tomoe) | 모니터링 & AI + 프론트 |


### **역할 분담**
### 👤 김민서 — 빌드 파이프라인
- 배포 요청 API 및 Lambda 로직 구현  
- CodeBuild 기반 Docker 이미지 빌드 자동화  
- ECR 이미지 저장 및 관리  

### 👤 백지은 — 배포 & 상태 관리
- App Runner 기반 자동 배포 구성  
- DynamoDB 상태 관리 및 이벤트 처리  
- Lambda를 통한 배포 상태 흐름 관리  

### 👤 이승현 — 모니터링 & AI + 프론트
- CloudWatch 기반 메트릭 및 로그 조회 기능 구현  
- Bedrock을 활용한 에러 로그 분석  
- React UI 및 사용자 인터페이스 개발  

### 🤝 공통 작업
- DynamoDB 스키마 설계  
- IAM 권한 구조 설계  
- API 인터페이스 정의  