# step2
<img width="334" height="183" alt="image" src="https://github.com/user-attachments/assets/f83420c8-2e8f-4d48-aa35-627628cf6e12" />
<br>import json - 도구 상자 가지고오기 <br>
import boto3 - 파이썬이 AWS 서비스(SQS,WAF 등)에게 명령할 때 사용하는 전용 도구(SDK)이다.<br>
그러니 sqs = boto3.client('sqs')에서 sqs라는 변수를 만들고, boto3.client에서 sqs도구를 가지고 오라는 코드를 작성했다. <br>
그다음 밑에 QUEUE_URL = 'SQS 주소'를 넣어 통신이 되도록 만들었다.<br>
테스트 박스를 눌러 Create new test event를 생성한다. 
<img width="597" height="414" alt="image" src="https://github.com/user-attachments/assets/02c87810-7598-4172-acc6-aedeb4e2b219" />
