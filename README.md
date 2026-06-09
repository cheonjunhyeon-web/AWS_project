# 서버리스 해킹 공격 실시간 탐지 및 자동 방어 
프로젝트 개요
AWS WAF에서 발생하는 보안 로그를 실시간으로 수집·분석하여,
위협 임계치를 초과하는 공격 IP를 자동으로 탐지하고 WAF 차단 목록에 동적으로 추가하는
서버리스 보안 자동화 시스템이다.
SQS를 통한 디커플링(Decoupling)과 Step Functions를 통한 상태 제어(State Management)를 핵심으로 설계했다.
관리자 개입 없이 탐지 → 분석 → 차단 → 기록 → 알림까지 전 과정이 자동으로 수행된다.

참여 인원: 1명 (개인 프로젝트)
수행 기간: 2026.02 ~ 2026.03 


아키텍처
<p align="center">
  <img src="images/architecture.png" width="800"/>
</p>
시스템은 탐지 → 분석 → 대응의 3단계 파이프라인으로 구성된다.

동작 흐름
1단계 — 탐지 및 수집
AWS WAF가 인바운드 HTTP/HTTPS 트래픽을 검사한다.
설정된 보안 규칙에 의해 차단된 트래픽의 메타데이터(IP, URI, Header 등)가 CloudWatch Logs로 실시간 전송된다.
AWS WAF → CloudWatch Logs (로그 그룹명: aws-waf-logs-*)
2단계 — 로그 추출 (Log Extractor Lambda)
CloudWatch Subscription Filter가 새로운 로그 스트림을 감지하면 Log Extractor Lambda를 트리거한다.
Lambda는 방대한 원본 JSON 로그에서 공격자 IP / 타임스탬프 / 매칭된 WAF 규칙 ID만 파싱하여 페이로드를 경량화한 뒤 SQS로 전송한다.
3단계 — 비동기 버퍼링 (SQS)
DDoS 등 대규모 트래픽 발생 시 Lambda 간 직접 동기 호출은 스로틀링(Throttling)을 유발한다.
SQS를 도입하여 생산자(Log Extractor)와 소비자(Threat Analyzer)의 결합도를 낮추는 디커플링 구조를 구현했다.

Queue Type: Standard Queue (높은 처리량)
Visibility Timeout: Threat Analyzer Lambda 처리 시간보다 길게 설정 (중복 처리 방지)

4단계 — 위협 분석 (Threat Analyzer Lambda)
Threat Analyzer Lambda가 SQS를 폴링하여 메시지를 가져온다.
동일 IP의 공격 빈도 / 페이로드 악성 수준을 기반으로 위협 점수(Risk Score)를 산출한다.
점수가 사전 정의된 임계치(Threshold)를 초과하면 Step Functions 상태 머신을 트리거한다.
Threat Analyzer Lambda가 Step Functions와 SQS 사이에 위치하여
위험 판정 시에만 Step Functions를 실행하는 게이트 역할 수행
5단계 — 자동 대응 오케스트레이션 (Step Functions)
Step Functions의 Parallel 상태로 아래 3개 Lambda가 동시에 실행된다.
Lambda역할연동 서비스Security Responder악성 IP를 WAF IP Set에 동적 추가 → 후속 트래픽 즉시 차단AWS WAF (wafv2:UpdateIPSet)DB-Logger탐지 IP / 공격 유형 / 조치 시간을 DynamoDB에 저장DynamoDB (Partition Key: Attacker_IP / Sort Key: Timestamp)SNS-Notifier관리자에게 위협 발생 및 차단 사실 알림 전송AWS SNS

기술 스택
분류서비스보안AWS WAF로그 수집Amazon CloudWatch Logs메시지 큐Amazon SQS (Standard Queue)컴퓨팅AWS Lambda (Python)오케스트레이션AWS Step Functions (Express Workflow)데이터베이스Amazon DynamoDB알림Amazon SNS

핵심 설계 포인트
SQS 디커플링
Log Extractor와 Threat Analyzer를 직접 연결하지 않고 SQS를 사이에 두어
대량 트래픽 발생 시에도 메시지 유실 없이 안정적으로 처리한다.
Step Functions 게이트 처리
Threat Analyzer가 위협 점수 판정 후 위험한 경우에만 Step Functions를 실행한다.
불필요한 Lambda 실행을 방지하여 비용을 절감하고,
복잡한 분기 처리와 재시도 로직을 Step Functions ASL(Amazon States Language)로 선언적으로 관리한다.
자동 방어 루프
Security Responder Lambda가 WAF IP Set API를 호출하여 식별된 악성 IP를 동적으로 추가한다.
이후 해당 IP의 모든 트래픽은 WAF 단에서 즉각 드롭(Drop)된다.
관리자 개입 없이 탐지부터 차단까지 자동으로 완결된다.

IAM 권한 구성
Lambda필요 권한Log Extractorlogs:GetLogEvents, sqs:SendMessageThreat Analyzersqs:ReceiveMessage, sqs:DeleteMessage, sqs:GetQueueAttributes, states:StartExecutionSecurity Responderwafv2:GetIPSet, wafv2:UpdateIPSetDB-Loggerdynamodb:PutItemSNS-Notifiersns:Publish
