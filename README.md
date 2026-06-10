# 서버리스 해킹 공격 실시간 탐지 및 자동 방어 시스템

> AWS 서버리스 서비스를 활용한 이벤트 기반 보안 자동화 파이프라인

<br>

## 📌 프로젝트 개요

AWS WAF에서 발생하는 보안 로그를 실시간으로 수집·분석하여,
위협 임계치를 초과하는 공격 IP를 자동으로 탐지하고 WAF 차단 목록에 동적으로 추가하는 서버리스 보안 자동화 시스템이다.

SQS를 통한 디커플링(Decoupling)과 Step Functions를 통한 상태 제어(State Management)를 핵심으로 설계했다.
관리자 개입 없이 **탐지 → 분석 → 차단 → 기록 → 알림** 전 과정이 자동으로 수행된다.

- 참여 인원: 1명 (개인 프로젝트)
- 수행 기간: 2026.02 ~ 2026.03

<br>

## 🏗️ 아키텍처

<p align="center">
<img width="1071" height="691" alt="제목 없는 다이어그램 drawio" src="https://github.com/user-attachments/assets/e633364b-74ae-4c08-8b6d-1b6c85036351" />


</p>

시스템은 **탐지 → 분석 → 대응**의 3단계 파이프라인으로 구성된다.

<br>

## ⚙️ 동작 흐름

### 1단계 — 탐지 및 수집

AWS WAF가 인바운드 HTTP/HTTPS 트래픽을 검사한다.
차단된 트래픽의 메타데이터(IP, URI, Header 등)가 CloudWatch Logs로 실시간 전송된다.
### 2단계 — 로그 추출 (Log Extractor Lambda)

CloudWatch Subscription Filter가 새로운 로그 스트림을 감지하면 Log Extractor Lambda를 트리거한다.
원본 JSON 로그에서 공격자 IP / 타임스탬프 / 매칭된 WAF 규칙 ID만 파싱하여 SQS로 전송한다.

### 3단계 — 비동기 버퍼링 (SQS)

DDoS 등 대규모 트래픽 발생 시 Lambda 간 직접 동기 호출은 스로틀링(Throttling)을 유발한다.
SQS를 도입하여 생산자(Log Extractor)와 소비자(Threat Analyzer)의 결합도를 낮추는 디커플링 구조를 구현했다.

- Queue Type: Standard Queue (높은 처리량)
- Visibility Timeout: Threat Analyzer Lambda 처리 시간보다 길게 설정 (중복 처리 방지)

### 4단계 — 위협 분석 (Threat Analyzer Lambda)

Threat Analyzer Lambda가 SQS를 폴링하여 메시지를 가져온다.
동일 IP의 공격 빈도 / 페이로드 악성 수준을 기반으로 위협 점수(Risk Score)를 산출한다.
점수가 사전 정의된 임계치를 초과하면 Step Functions 상태 머신을 트리거한다.

> Threat Analyzer가 SQS와 Step Functions 사이에서 게이트 역할을 수행하여
> 위험 판정 시에만 Step Functions를 실행한다.

### 5단계 — 자동 대응 오케스트레이션 (Step Functions)

Step Functions의 Parallel 상태로 아래 3개 Lambda가 동시에 실행된다.

| Lambda | 역할 | 연동 서비스 |
|---|---|---|
| Security Responder | 악성 IP를 WAF IP Set에 동적 추가 → 후속 트래픽 즉시 차단 | AWS WAF (wafv2:UpdateIPSet) |
| DB-Logger | 탐지 IP / 공격 유형 / 조치 시간 저장 | DynamoDB |
| SNS-Notifier | 관리자에게 위협 발생 및 차단 알림 전송 | AWS SNS |

<br>

## 🛠️ 기술 스택

| 분류 | 서비스 |
|---|---|
| 보안 | AWS WAF |
| 로그 수집 | Amazon CloudWatch Logs |
| 메시지 큐 | Amazon SQS (Standard Queue) |
| 컴퓨팅 | AWS Lambda (Python) |
| 오케스트레이션 | AWS Step Functions (Express Workflow) |
| 데이터베이스 | Amazon DynamoDB |
| 알림 | Amazon SNS |

<br>

## 💡 핵심 설계 포인트

**SQS 디커플링**

Log Extractor와 Threat Analyzer를 직접 연결하지 않고 SQS를 사이에 두어
대량 트래픽 발생 시에도 메시지 유실 없이 안정적으로 처리한다.

**Step Functions 게이트 처리**

Threat Analyzer가 위협 점수 판정 후 위험한 경우에만 Step Functions를 실행한다.
불필요한 Lambda 실행을 방지하여 비용을 절감하고,
복잡한 분기 처리와 재시도 로직을 ASL(Amazon States Language)로 선언적으로 관리한다.

**자동 방어 루프**

Security Responder Lambda가 WAF IP Set API를 호출하여 악성 IP를 동적으로 추가한다.
이후 해당 IP의 모든 트래픽은 WAF 단에서 즉각 드롭(Drop)된다.
관리자 개입 없이 탐지부터 차단까지 자동으로 완결된다.

<br>

## 🔐 IAM 권한 구성

| Lambda | 필요 권한 |
|---|---|
| Log Extractor | `logs:GetLogEvents` `sqs:SendMessage` |
| Threat Analyzer | `sqs:ReceiveMessage` `sqs:DeleteMessage` `states:StartExecution` |
| Security Responder | `wafv2:GetIPSet` `wafv2:UpdateIPSet` |
| DB-Logger | `dynamodb:PutItem` |
| SNS-Notifier | `sns:Publish` |

<br>

## ⚠️ 기술적 한계

**1. DDoS 방어의 근본적 한계**

현재 시스템은 공격자 IP를 WAF IP Set에 개별적으로 추가하는 방식으로 차단한다.
실제 DDoS 공격은 수천~수만 개의 IP에서 동시에 발생하기 때문에 IP 개별 차단만으로는 효과적인 방어가 어렵다.
LockToken 재시도 로직으로 동시성 충돌을 해결했지만, 대규모 공격 시 WAF API 호출 제한에 도달할 수 있다.

**2. 전통적 공격 패턴 중심**

SQL Injection, XSS 등 전통적인 웹 공격 탐지에 집중되어 있어,
2026년 클라우드 환경에서 증가하는 최신 공격 기법에 대한 대응이 부족하다.

<br>

## 🚀 향후 개선 방향

**1. Honeypot 기반 트래픽 우회 전략**

DDoS 트래픽을 가짜 목적지로 우회시켜 공격자는 성공했다고 착각하지만 실제 서비스는 무중단으로 유지하는 방어 전략 구현

- Route 53 Health Check 연동: Threat Analyzer에서 DDoS 탐지 시 DNS 레코드를 Honeypot 서버로 변경
- Honeypot EC2: 가짜 응답을 반환하는 경량 Nginx 서버 구축
- 자동 복구 Lambda: 5분 후 자동으로 DNS를 원래 서버로 복구
- 트래픽 분석: Honeypot 유입 트래픽 패턴 분석으로 공격 특성 학습

**2. 최신 공격 패턴 대응**

- LLM Injection 탐지: AI 모델 대상 악의적 프롬프트 삽입 공격 탐지 로직 추가
- Event Injection 방어: S3, SNS 이벤트 조작 공격 방어를 위한 소스 검증 및 서명 확인 로직 추가
- IAM Abuse 방지: 최소 권한 원칙에 따른 Lambda 권한 세밀 조정

<br>

## 📊 성과 및 결론
<img width="45%" alt="스크린샷 2026-06-10 오전 9 44 37" src="https://github.com/user-attachments/assets/e3e5fe07-77fb-4e6c-9c77-1027e5432a13" />
<img  width="45%" alt="스크린샷 2026-06-10 오전 9 44 19" src="https://github.com/user-attachments/assets/1e56887c-3459-4e5b-9862-f3af6039ab92" />

500개 공격 샘플로 DDoS 시뮬레이션을 진행했을 때,
**500+ 요청/분을 정확히 탐지하고 WAF 차단 / 이메일 알림 / DB 저장이 병렬로 자동 실행**되는 것을 확인했다.

이 프로젝트는 CWPP 앞단에 1차 방어선을 구축하여 SQL Injection, XSS 같은 전통적인 웹 공격을 사전에 필터링함으로써,
전체 보안 인프라의 트래픽 부하를 분산하는 효과를 검증했다.

- AWS Lambda 기반 서버리스 아키텍처의 동작 원리 및 이벤트 기반 실행 모델 이해
- Step Functions Parallel 상태를 활용한 병렬 처리 오케스트레이션 구현 경험
- LockToken 재시도 로직을 통한 WAF API 동시성 충돌 해결 경험
