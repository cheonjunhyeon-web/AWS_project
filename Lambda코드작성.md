# step2
<img width="334" height="183" alt="image" src="https://github.com/user-attachments/assets/f83420c8-2e8f-4d48-aa35-627628cf6e12" />
<br>import json - 도구 상자 가지고오기 <br>
import boto3 - 파이썬이 AWS 서비스(SQS,WAF 등)에게 명령할 때 사용하는 전용 도구(SDK)이다.<br>
그러니 sqs = boto3.client('sqs')에서 sqs라는 변수를 만들고, boto3.client에서 sqs도구를 가지고 오라는 코드를 작성했다. <br>
그다음 밑에 QUEUE_URL = 'SQS 주소'를 넣어 통신이 되도록 만들었다.<br>
테스트 박스를 눌러 Create new test event를 생성한다. <br>
<img width="985" height="643" alt="image" src="https://github.com/user-attachments/assets/80be2542-9982-4c4a-9bb4-1b6c417cae14" />
정상적으로 통신이 되는지 테스트를 해봤다. 
<img width="1179" height="420" alt="image" src="https://github.com/user-attachments/assets/14b2071c-6b1d-48c4-a66f-31cb65cea0d0" />
정상적으로 통신이 되었다는 문구가 나왔다. SQS에서도 메시지가 뜨는지 확인해준다. 
<img width="1581" height="486" alt="image" src="https://github.com/user-attachments/assets/1de46ce0-49af-4743-8772-73554146f5e5" />
메시지 풀링으로 들어가면 수신하였던 메시지 로그를 확인할 수 있다. 뜨는거 보니 잘 작동한다.<br>
일단 1번 로그확인 람다는 통신이 되니 이상태로 놔두고 두번째 람다를 만들어 준다. 2번째 람다를 다 만들고나서 첫번째 람다를 연동용으로 코드를 변경시킬 것이다. <br>
