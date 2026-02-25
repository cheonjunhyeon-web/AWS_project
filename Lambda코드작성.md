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

# step3
Lambda 코드 작성 
<img width="924" height="123" alt="image" src="https://github.com/user-attachments/assets/a006dee7-e65c-4bfd-aea9-2618119d9b53" /><br>
def lambda_handler로 지정 안에 event와 context로 바구니 생성해준다. 이때 event는 무엇을 처리할 것인지(공격 데이터, 로그내용)외부 정보들을 담는 역할을 한다 <br>
context는 어떻게, 어디서 실행되는 가(남은시간, 함수이름, 메모리 크기) 등등 내부상황을 담는 역할을 한다. <br>
- try :  and except  , try는 말그대로 시도하는 것이다.<br> 하지만 try안에 있는 코드들은 감시 구역으로 에러가 나는지 지켜본다. 만약 에러가 나면 except로 바로 넘어가서 에러가 처리된다 <br> 
- for record in event.get('Records', []): <br>
- for 은 반복문,  event.get('Records')는 내용물이 들어있는 전체 목록, 꺼낸 내용물을 담은 이름표(변수)가 record <br>
- ** event.get('Record')에 Record로 하는 이유는 SQS는 람다에게 데이터를 던져줄 때, 봉투 겉면에 무조건 Records라고 이름을 써서 보내기로 정의되어 있다. 만약 내가 임의로 mmmm이라는 이름으로 정의하면
SQS가 보낸 Records를 못 본체하고 지나치게 된다. **<br>
- 다음으로 [], 만약 []가 없다면 Records를 찾다 데이터가 없으면 다음 코드로 넘어가는 것이 아닌, 에러를 내며 중단을 내게된다. 부드럽게 대처하기 위해 []를 넣어 빈 리스트를 줘서 코드를 넘어가게 만들어준다. <br><br>
### message_body = json.loads(record['body'])
람다가 SQS로 데이터를 보낼 때, 모든 정보는 문자열(string)이라는 단순한 글자 덩어리로 변해버린다. <br>
변환 전: (record['body']): "{ip: "1.2.3.4", "attack": "SQL"}" (그냥 긴 한줄로 인식) <br>
json.loads은 글자를 데이터로 변환해준다. <br>
여기서 loads는 Load String(문자열을 불러오다)의 줄임말이다. <br>
뭉쳐있는 글자를 잘 읽어보고, 파이썬이 좋아하는 딕셔너리(Distionary)구조로 다시 조립해 라는 뜻이다. <br>
변환 후 message_body['ip']라고 부르면 "1.2.3.4"를 바로 뽑아줄 수 있다. <br><br>
<img width="628" height="102" alt="image" src="https://github.com/user-attachments/assets/21055606-de5a-42ac-8e89-a467ee541be9" /><br>
여기는 쉽다. attacker_ip 와 attack_type 이라는 변수를 각각 만들고 message_body에서 데이터를 가지고 오는 코드이다. <br>만약 데이터에 ip라는 항목이 없으면, 에러 내지 말고 대신 Unknown(알수없음)이라고 적어줘. 라는 코드이다. <br><br>
<img width="637" height="245" alt="image" src="https://github.com/user-attachments/assets/9e1a8add-52f7-44e4-8693-fc432a8826c7" />
<br> 만약 SQL Injection 타입이나 Cross-Site Scripting 타입이면 위험 수위를 크리티컬로 지정하고 is_danger = True로 해준다. <br>
is_danger이 True로 적여있다면, Step Functions를 깨워서 실제로 IP를 차단하게 된다. False라면 아무일도 일어나지 않고 평화롭게 로그만 남고 끝남. <br>
** 위쪽 if에서 True가 되었다면 밑에 if에서 위험 수준으로 확정 짓고, 위쪽 if에서 False가 나온다면 밑에 if에서 else로 구동된다. ** <br>
<img width="653" height="222" alt="image" src="https://github.com/user-attachments/assets/0ab0b337-9ce3-4869-a39c-300bfa3287b6" />
<br> 람다가 성공했는지 statusCode를 보여주게 만드는 코드 , body에 구체적인 결과물 즉, 차단된 IP정보 등을 여기에 담아서 보낸다. json.dumps()는 이전에 글자를 데이터로 변환했던 것을 반대로 데이터를 글자로 포장해서 내보냄.
<img width="1177" height="393" alt="image" src="https://github.com/user-attachments/assets/53cf8c33-e343-4795-89a8-cb21f4291f4a" />
테스트 결과 위험순위를 알려주고, step functiond을 자동으로 가동시킬 수 있도록 만들었다. 다음으로 Step functions을 설정할 차례이다. <br>


## step4<br>
lambda2 코드 수정 <br>
<img width="387" height="69" alt="image" src="https://github.com/user-attachments/assets/265d01e5-3a09-4b09-8172-4b021d3139e9" /><br>
일단 boto3에 있는 stepfunctions 도구를 불러오기 위해 상단에 코드 작성을 한다. <br>
<img width="800" height="243" alt="image" src="https://github.com/user-attachments/assets/b6434ab5-1c45-480f-b2c4-23bbd4db0676" /><br>
위험 등급일 경우 lambda2에서 step functions를 불러올 수 있게 만들어 줘야한다.<br>
respons라는 변수에서 나의 step functions의 경로를 붙여넣어준다. <br>
input을 사용하는 이유는 step Functions로 넘어갈때 그냥 깨우는 것이 아닌, 어떤 공격이 들어왔는지 정보를 같이 줘야한다. <br>그 정보를 담는 파라미터의 이름이 input. <br>
json.dumps : 데이터를 전송용 문자열로 포장하는 과정 <br>
파이썬의 딕셔너리 데이터는 메모리상에만 있는 복잡한 덩어리라 그대로 전송할 수 없다. json.dumps는 이 복잡한 데이터를 하나의 긴 글자(string)로 변환해 준다. <br>

## step5
SNS-lambda 코드 
<img width="963" height="696" alt="image" src="https://github.com/user-attachments/assets/6ca8c81a-86c6-45f7-b2e0-bfc26e6445e6" />
메일에 보낼 정보들을 꺼내와야 한다. 그렇기에 이런 lambda2에서 썻던 attacker_ip라던가 attack_type등을 여기로 불러와주면 된다.  <br>
 <img width="587" height="224" alt="image" src="https://github.com/user-attachments/assets/107618f0-34f3-4b33-8fee-29d6d5432c45" />
<br>
이전 message는 보낼 편지를 작성했다면 publish에서는 우체국 역할을 한다. 내가 보낼려고하는 sns 주체의 주소와 어떤 메시지를 보낼지를 선택하여 보내주면 된다. 

## step 6
DB-lambda 코드<br> 
<img width="726" height="450" alt="image" src="https://github.com/user-attachments/assets/01a0887f-86fc-4e8c-a571-60f96d1367a4" />
<br>
- import time, uuid 
<br> 데이터가 언제 쌓였는지(시간),그리고 각 데이터에 고유한 이름표(ID)를 붙여주기 위해 필요한 도구<br>
- Table = ...('Security-DB')는 DynamoDB에 만들어둔 테이블인 Security-DB를 딱 집어서 연결하는 것이다 <br>
     table.put_item(<br>
          Item={<br>
               'attack_id':str(uuid.uuid4()),<br>
               'timestamp':int(time.time()),<br>
               'attacker_ip': ip,<br>
               'attack_type': attack,<br>
               'threat_level': level<br>
          }<br>
     )<br>
     이부분에서 table은 테이블 장부 자체를 의미하고, .put_item은 새로운 빈 줄에 내용을 적는 부분이다. <br>
     Item{}부분은 실제로 적을 구체적인 내용이다.
  
