# AWS Security Automation Project

> **AWS 기반 실시간 보안 탐지 · 알림 · 자동대응(SOAR) 시스템 구축 프로젝트에서  
> 제가 직접 설계하고 구현한 부분만 정리한 포트폴리오 문서입니다.**

본 프로젝트는 AWS EventBridge, Lambda, CloudTrail, DynamoDB, API Gateway WebSocket 등을 활용해  
**보안 이벤트 실시간 탐지 → 알림 → 자동대응(Quarantine / Block / Logging)** 까지 수행하는  
SOAR 스타일의 자동화 시스템을 구축한 팀 프로젝트입니다.

> 원본 프로젝트 레포:  
👉 https://github.com/TeamLayer3/AWS-Security-Automation-Project


---

# 📌 1. 프로젝트 개요

AWS 환경에서 발생하는 주요 보안 이벤트를 자동으로 탐지하고 즉시 조치하는  
**End-to-End 자동 대응 플랫폼**입니다.

### 구축 목표
- CloudTrail 기반의 실시간 탐지
- EventBridge를 통한 시나리오 별 분석 처리
- Lambda 기반 자동 대응 업무 구현
- DynamoDB & WebSocket 기반 실시간 알림 대시보드 구축
- EC2/DVWA 서버의 이상 트래픽 탐지 및 차단

---

# 📌 2. 제가 담당한 핵심 역할 (요약)

프로젝트에서 제가 직접 개발/설계한 부분은 아래와 같습니다.

### 1) 탐지 Lambda 함수 개발
- Impossible Travel Login (불가능한 위치 로그인)
- CloudTrail Tampering 시도 감지
- DVWA 스캐너 이상 트래픽 감지 (4만 건 이상 요청 패턴)
- SSH World-Open(0.0.0.0/0) 규칙 생성 감지
- 잘못된 IAM 사용/AccessKey 기반 탐지

### 2) 자동 대응 플레이북 구현
- Open SSH 감지 → 보안 그룹 Quarantine 자동 적용  
- DVWA 공격 감지 → HTTP 차단 + 보안그룹 교체  
- CloudTrail Tamper → 로그 스냅샷 S3 저장
- 고위험 이벤트 종합 처리 → Incident 기록 및 실시간 송신

### 3) WebSocket 실시간 알림 구조 설계
- API Gateway WebSocket 연결 관리 테이블 설계
- Alert/Action 채널 분리 구조 설계 (EVENTS vs ACTIONS)
- Lambda에서 Dashboard로 Incident 구조화 전송

### 4) DynamoDB 상태 테이블 설계
- TTL 기반 Sliding Window 탐지 구조
- 멱등성(Idempotency) 보장을 위한 EventID 기반 저장

### 5) Dashboard 알림 포맷 개선
- 기존 undefined 값 제거
- incident 구조화 필드(source, region 등) 통일
- 알림 UX를 제품 형태로 정리

---

# 📌 3. 전체 아키텍처 (개인 정리본)

flowchart TD

A[CloudTrail / VPC Flow / Config / Scanner Logs] --> B(EventBridge Rules)

B --> C1[Detection Lambda Functions]
C1 --> D1[DynamoDB - State Table (Sliding Window + Idempotency)]
C1 --> D2[DynamoDB - WebSocket Connections]

%% Real-time Alert Path
C1 -->|Structured Incident JSON| E[API Gateway WebSocket]
E --> F[Real-Time Security Dashboard]

%% Auto Remediation Path
C1 -->|Trigger Action| G[Remediation Lambda Functions]
G --> H1[Modify Security Group (Quarantine)]
G --> H2[Block HTTP / Ingress Rules]
G --> H3[Archive Logs to S3]

---

# 📌 4. 내가 구현한 주요 탐지 로직 상세

## 1) Impossible Travel Login Detection

* 최근 로그인 IP Geo 기반 위치 기록
* 이전 로그인 위치 vs 신규 위치 거리 계산
* 이동 속도가 비정상일 경우 FLAG

## 2) CloudTrail Tampering Detection

* StopLogging / DeleteTrail / UpdateTrail 이벤트 감시
* 로그인 직후 60초 간의 window 내부 tamper 시 HIGH RISK 처리

## 3) DVWA 공격 스캐너 탐지

* 5분 내 요청 개수가 임계값(40,000+) 초과 시 공격으로 판단
* User-Agent 기반 스캐너 패턴 regex 분석

## 4) SSH World-Open 감지

* 22/TCP + 0.0.0.0/0 + ::/0 규칙 생성 여부 모니터링
* 반복/다중 시도 시 가중치 기반 위험 레벨 증가

---

# 📌 5. 자동 대응(Playbook) 상세

## 🟥 1) OpenSSH World-Open → EC2 Quarantine

* 위험 규칙 감지 → Quarantine SG 자동 부착
* 기존 SG 회수 및 ALB 라우팅 재검증

## 🟧 2) HTTP 공격 감지 → 즉시 Ingress 차단

* DVWA 공격 상황 감지 시 SG 수정
* WebSocket 대시보드에 실시간 "차단" 알림 전송

## 🟦 3) CloudTrail Tamper → 로그 아카이빙

* tamper 이벤트 발생 시 CloudWatch 로그 S3 저장
* Incident 테이블 기록 + 알림 전송

---

# 📌 6. Incident 구조 (Dashboard로 전송된 데이터)

```json
{
  "type": "remediation",
  "source": "ec2",
  "region": "us-east-1",
  "action": "QuarantineInstance",
  "target": "i-0ac2cbc9d6a8afc46",
  "status": "EXECUTED",
  "time": "2025-11-28T08:30:00Z"
}
```

---

# 📌 7. 문제 해결 경험 (Troubleshooting)

### ⚠ WebSocket undefined 문제 해결

* Lambda 이벤트 구조 통일
* incident builder 함수 설계
* note 필드 제거, source/region 정상화

### ⚠ DynamoDB Hot Partition 이슈 가능성 완화

* TTL + pk 분리
* Sliding Window 방식 개선

### ⚠ 자동 대응 오작동

* Actor ARN / Window 기반 룰 필터링 추가
* Duplicate Event Skip 로직 적용

---

# 📌 8. 기술 스택

| 구분         | 기술                                   |
| ---------- | ---------------------------------------- |
| Cloud      | AWS EC2, Lambda, EventBridge, CloudTrail |
| Storage    | DynamoDB, S3                             |
| Networking | Security Group, VPC                      |
| Real-time  | API Gateway WebSocket                    |
| Infra      | IAM, KMS                                 |
| Language   | Python 3.x                               |
| Tools      | GitHub, CloudShell, PyCharm              |

---

# 📌 9. 원본 팀 프로젝트 링크

👉 [https://github.com/TeamLayer3/AWS-Security-Automation-Project](https://github.com/TeamLayer3/AWS-Security-Automation-Project)

---

# 📌 10. About Me

**윤서원(Seowon Yoon)**
Seeking : Cloud Security / Security Automation / SOC / DevSecOps
Email : lucielyoon1129@naver.com / seowon6766@gmail.com
GitHub : [https://github.com/Wonyoon-Luciel](https://github.com/Wonyoon-Luciel)
velog : https://velog.io/@seowon6766
