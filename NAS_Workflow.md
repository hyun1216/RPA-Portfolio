📂 NAS & Node-RED 기반 이벤트 감지 시스템 (Event-Driven Automation)
Summary: Synology NAS와 Node-RED를 활용하여, 담당자가 파일을 업로드하는 즉시 자동으로 RPA 프로세스가 시작되는 '이벤트 기반(Event-Driven)' 워크플로우를 구축했습니다.


1. 🔄 전체 워크플로우 (Workflow Diagram)
NAS의 특정 폴더를 24시간 감시하며, 파일 이벤트 발생 시 아래와 같은 흐름으로 처리됩니다.

[프로세스 상세 로직]
24/7 Watch Dog (상시 감지):

Node-RED의 watch 노드가 NAS 마운트 폴더의 파일 생성을 실시간으로 감지

유효성 검증 (Validation):

업로드된 파일이 정해진 양식(파일명, 확장자 등)인지 1차 검증 수행

오류 시: 담당자에게 "양식 오류" 알림 즉시 발송

RPA 트리거 & 시작 알림:

검증 통과 시 RPA 봇(Power Automate Desktop) 호출 (Webhook 방식)

동시에 담당자에게 **텔레그램(Telegram)**으로 "작업이 시작되었습니다" 알림톡 전송

작업 수행 (RPA):

이메일 발송 및 데이터 처리 작업 수행

완료 피드백 (Feedback):

RPA가 결과 파일을 '완료 폴더'에 업로드

Node-RED가 이를 다시 감지하여 담당자에게 "작업 최종 완료" 텔레그램 전송

2. 💻 Node-RED 핵심 플로우 코드 (JSON)
실제 운영 중인 Node-RED 플로우의 핵심 로직입니다. (보안을 위해 일부 정보는 마스킹 처리되었습니다.)

<details> <summary>👇 (Click) Node-RED JSON 코드 펼쳐보기</summary>

```json [ { "id": "nas_watcher", "type": "watch", "z": "flow_1", "name": "NAS 폴더 감지(Input)", "files": "/mnt/nas/input_data", "recursive": false, "x": 120, "y": 80, "wires": [["file_validator"]] }, { "id": "telegram_sender", "type": "telegram sender", "z": "flow_1", "name": "텔레그램 알림톡 발송", "bot": "telegram_config_bot", "haserroroutput": false, "outputs": 1, "x": 580, "y": 240, "wires": [[]] }, { "info": "※ 이 코드는 예시입니다. 실제 새봄이의 Node-RED JSON 코드를 여기에 붙여넣으세요!" } ] ```

</details>


3. 📸 실제 구동 화면 (Demo)
