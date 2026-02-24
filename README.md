# 서버리스 해킹 공격 실시간 탐지 및 자동 방어 AI
202-02-24일부터 시작  
총 3개의 람다를 생성하여 로그를 읽고 해킹인지 판단한 뒤 방화벽 설정을 자동으로 바꿔서 공격자를 차단하는 방식의 프로젝트를 만들었다.  
1. 공격 발생 및 증거 수집(Detection)
   
    AWS WAF(방화벽)나 CloudWatch가 수상한 움직임을 포착하고 로그를 남긴다. 이 로그에는 해커의 IP, 공격 시간, 공격 패턴이 담겨있다.
3. 분석(Analyzer)
   
   이 로그가 단순한 실수인지 해킹인지를 판단한다.
5. 판단
   
   분석 결과가 위험하다고 판단되면 두 번째 람다가 실행된다. (여기서 Step Function가 명령을 직접 내리는 지휘자 역할을 한다)
7. 결과 보고 및 알림
   
   마지막 람다가 동작하여 리포트를 DynamoDB에 저장하고, 휴대폰으로 알림을 보낸다.

## Architecture
<img width="1246" height="698" alt="image" src="https://github.com/user-attachments/assets/481ce715-f4ab-492c-87be-0ebf69b862c4" />
잘못 만들었다. 현재 threat-Analyzer이 step Functions안에 있다 이것은 틀린 부분이다. <br>
threat-Analyzer이 SQS와 Step Functions사이에 위치해야하며, threat-Analyzer이 위험한지 아닌지 판단하고, 위험한 순간에만 Step Functions를 가동하여,
Step Functions에 있는 람다 3개가 동작해야한다. 안에 있는 람다는 각각 차단/알림/기록 순이다. 
<img width="1089" height="699" alt="image" src="https://github.com/user-attachments/assets/ef630441-5d5a-4fc6-a321-88c0abcb3c12" />


