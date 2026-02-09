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

```[
    {
        "id": "a1f80dcd91645bf8",
        "type": "tab",
        "label": "Syslog 파일 생성 감시 (에러수정)",
        "disabled": false,
        "info": ""
    },
    {
        "id": "7fb3f98e5fc04da4",
        "type": "syslog-input2",
        "z": "a1f80dcd91645bf8",
        "name": "NAS syslog",
        "socktype": "udp",
        "address": "0.0.0.0",
        "port": "20514",
        "topic": "",
        "x": 110,
        "y": 300,
        "wires": [
            [
                "5463167ccbfcd663"
            ]
        ]
    },
    {
        "id": "5463167ccbfcd663",
        "type": "switch",
        "z": "a1f80dcd91645bf8",
        "name": "로그 종류 분기",
        "property": "payload.hostname",
        "propertyType": "msg",
        "rules": [
            {
                "t": "eq",
                "v": "FileStation",
                "vt": "str"
            },
            {
                "t": "else"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 2,
        "x": 300,
        "y": 300,
        "wires": [
            [
                "442f946d0e7d1584",
                "bcc9d69b390c5223"
            ],
            [
                "5af2bf169a2df72f",
                "bb878ded58028f1f"
            ]
        ]
    },
    {
        "id": "5af2bf169a2df72f",
        "type": "function",
        "z": "a1f80dcd91645bf8",
        "name": "SMB 전용 필터링",
        "func": "const log = msg.payload && msg.payload.msg ? msg.payload.msg : \"\";\nif (!log.toLowerCase().includes(\".xls\")) return null;\n\n// 1) SMB 전용: 파일 쓰기(write) 작업인지 확인\nif (!log.toLowerCase().includes(\"write\")) return null;\n\n// 2) 임시 파일(~$...) 제외\nif (log.match(/~\\$.*\\.xlsx?/i)) return null;\n\n// 3) 경로 추출\nlet match = log.match(/Path:\\s*(.*?\\.xlsx?),/i);\nif (!match || !match[1]) return null;\n\n// ★★★ 4) [버그 수정] 확장자 앞 띄어쓰기 박멸 + 양끝 공백 제거 ★★★\n// .trim()은 전체 양끝을, .replace()는 .xls 바로 앞의 공백(\\s+)만 찾아 지워줘!\nlet filePath = match[1].trim().replace(/\\s+\\.xls/i, '.xls');\n\n// 5) '샵라이프' 폴더 확인\nif (!filePath.includes(\"샵라이프\")) return null;\n\n// ★★★ 6) [신규 기능] '결과파일' 폴더 여부 체크 ★★★\n// 경로에 '결과파일'이 들어있으면 true, 아니면 false를 저장해!\nmsg.isResultFile = filePath.includes(\"결과파일\");\n\n// 7) 최종 경로 및 토픽 설정\nmsg.payload = \"\\\\\\\\192.168.0.8\" + filePath.replace(/\\//g, \"\\\\\");\nmsg.topic = \"[NAS] \" + msg.payload.split(\"\\\\\").pop();\n\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 530,
        "y": 340,
        "wires": [
            [
                "ed39dc27c3ab1ff0",
                "348f776979c3e847"
            ]
        ]
    },
    {
        "id": "442f946d0e7d1584",
        "type": "function",
        "z": "a1f80dcd91645bf8",
        "name": "웹 전용 필터링",
        "func": "const log = msg.payload && msg.payload.msg ? msg.payload.msg : \"\";\nif (!log.toLowerCase().includes(\".xls\")) return null;\n\n// 1) 액션 체크 (웹 업로드 및 쓰기 작업 포함)\nconst webActions = [\"upload\", \"create\", \"move\", \"write\"];\nconst isValid = webActions.some(a => log.toLowerCase().includes(a));\nif (!isValid) return null;\n\n// 2) 임시 파일 제외\nif (log.match(/~\\$.*\\.xlsx?/i)) return null;\n\n// 3) 경로 추출\nlet match = log.match(/Path:\\s*(.*?\\.xlsx?),/i);\nif (!match || !match[1]) return null;\n\n// ★★★ 4) [버그 수정] 띄어쓰기 박멸 + 양끝 공백 제거 ★★★\n// .trim()으로 양끝을 잡고, .replace()로 확장자 바로 앞의 공백을 지워버려!\nlet filePath = match[1].trim().replace(/\\s+\\.xls/i, '.xls');\n\n// 5) '샵라이프' 폴더 확인\nif (!filePath.includes(\"샵라이프\")) return null;\n\n// ★★★ 6) [신규 기능] '결과파일' 폴더 여부 체크 ★★★\n// 경로에 '결과파일'이 포함되어 있으면 true, 아니면 false를 담아줘\nmsg.isResultFile = filePath.includes(\"결과파일\");\n\n// 7) 최종 경로 및 토픽 설정\nmsg.payload = \"\\\\\\\\192.168.0.8\" + filePath.replace(/\\//g, \"\\\\\");\nmsg.topic = \"[NAS] \" + msg.payload.split(\"\\\\\").pop();\n\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 530,
        "y": 260,
        "wires": [
            [
                "ed39dc27c3ab1ff0",
                "348f776979c3e847"
            ]
        ]
    },
    {
        "id": "348f776979c3e847",
        "type": "delay",
        "z": "a1f80dcd91645bf8",
        "name": "",
        "pauseType": "delay",
        "timeout": "3",
        "timeoutUnits": "seconds",
        "rate": "1",
        "nbRateUnits": "1",
        "rateUnits": "second",
        "randomFirst": "1",
        "randomLast": "5",
        "randomUnits": "seconds",
        "drop": false,
        "allowrate": false,
        "outputs": 1,
        "x": 780,
        "y": 180,
        "wires": [
            [
                "switch-mail-check"
            ]
        ]
    },
    {
        "id": "f47b807c9e2a40bf",
        "type": "e-mail",
        "z": "a1f80dcd91645bf8",
        "server": "smtp.gmail.com",
        "port": "465",
        "authtype": "BASIC",
        "saslformat": false,
        "token": "oauth2Response.access_token",
        "secure": true,
        "tls": true,
        "name": "shoplife@taxshop.co.kr",
        "dname": "메일 알림 (jkzc dskl dvfz okoi)",
        "x": 1220,
        "y": 180,
        "wires": []
    },
    {
        "id": "ed39dc27c3ab1ff0",
        "type": "function",
        "z": "a1f80dcd91645bf8",
        "name": "데이터추출",
        "func": "var fullPath = msg.payload; \nvar pathParts = fullPath.split(/[/\\\\]/); \nvar managerName = pathParts[pathParts.length - 2]; \nvar regex = /([A-Za-z]{2}-\\d{5})/; \nvar match = fullPath.match(regex);\nvar projectCode = match ? match[0] : \"코드없음\";\nvar now = new Date().toLocaleString();\n\nmsg.payload = {\n    \"담당자\": managerName,\n    \"프로젝트코드\": projectCode,\n    \"파일명\": pathParts[pathParts.length - 1],\n    \"시간\": now\n};\nmsg.managerName = managerName;\nmsg.projectCode = projectCode;\nmsg.fullPath = fullPath;\n\nreturn msg;",
        "outputs": 1,
        "x": 770,
        "y": 300,
        "wires": [
            [
                "55f8a1ae5a0bd2fb",
                "6596bc933168071a",
                "14e9e95a14139990"
            ]
        ]
    },
    {
        "id": "55f8a1ae5a0bd2fb",
        "type": "function",
        "z": "a1f80dcd91645bf8",
        "name": "텔레그램 메세지 포맷",
        "func": "var text = `[RPA 알림] 파일이 도착했습니다!\\n\\n`;\ntext += `👤 담당자: ${msg.managerName}\\n`;\ntext += `🏷️ 코드: ${msg.projectCode}\\n`;\ntext += `📂 경로: ${msg.fullPath}`;\n\nmsg.payload = {\n    chatId: 8566205095,\n    type: 'message',\n    content: text\n};\nreturn msg;",
        "outputs": 1,
        "x": 1000,
        "y": 300,
        "wires": [
            [
                "6b20458de67f7f0e"
            ]
        ]
    },
    {
        "id": "6b20458de67f7f0e",
        "type": "telegram sender",
        "z": "a1f80dcd91645bf8",
        "bot": "e6cec9db5cd72f78",
        "outputs": 1,
        "x": 1210,
        "y": 300,
        "wires": [
            []
        ]
    },
    {
        "id": "bcc9d69b390c5223",
        "type": "debug",
        "z": "a1f80dcd91645bf8",
        "name": "WEB 로그 받기",
        "active": true,
        "complete": "true",
        "x": 520,
        "y": 140,
        "wires": []
    },
    {
        "id": "bb878ded58028f1f",
        "type": "debug",
        "z": "a1f80dcd91645bf8",
        "name": "SMB 로그 받기",
        "active": true,
        "complete": "true",
        "x": 520,
        "y": 420,
        "wires": []
    },
    {
        "id": "14e9e95a14139990",
        "type": "debug",
        "z": "a1f80dcd91645bf8",
        "name": "OK: 로그 확인",
        "active": true,
        "complete": "true",
        "x": 980,
        "y": 120,
        "wires": []
    },
    {
        "id": "6596bc933168071a",
        "type": "function",
        "z": "a1f80dcd91645bf8",
        "name": "대시보드 기록저장",
        "func": "var logList = flow.get(\"rpaLogList\") || [];\nvar now = new Date();\nnow.setHours(now.getHours() + 9); \nvar timeString = now.getFullYear() + \".\" + (now.getMonth()+1) + \".\" + now.getDate() + \". \" + now.getHours() + \":\" + now.getMinutes();\n\nlogList.unshift({\n    \"시간\": timeString,\n    \"담당자\": msg.managerName,\n    \"코드\": msg.projectCode,\n    \"경로\": msg.fullPath\n});\nif (logList.length > 30) logList.pop();\nflow.set(\"rpaLogList\", logList);\nmsg.payload = logList;\nreturn msg;",
        "outputs": 1,
        "x": 1010,
        "y": 420,
        "wires": [
            [
                "8cb6fed9b4bbf896"
            ]
        ]
    },
    {
        "id": "8cb6fed9b4bbf896",
        "type": "ui_table",
        "z": "a1f80dcd91645bf8",
        "group": "ed5cb154bf41a709",
        "name": "RPA현황",
        "columns": [
            {
                "field": "시간",
                "title": "시간"
            },
            {
                "field": "담당자",
                "title": "담당자"
            },
            {
                "field": "코드",
                "title": "프로젝트코드"
            }
        ],
        "outputs": 0,
        "x": 1220,
        "y": 420,
        "wires": []
    },
    {
        "id": "switch-mail-check",
        "type": "switch",
        "z": "a1f80dcd91645bf8",
        "name": "메일 발송 체크",
        "property": "isResultFile",
        "propertyType": "msg",
        "rules": [
            {
                "t": "false"
            }
        ],
        "checkall": "true",
        "repair": false,
        "outputs": 1,
        "x": 1000,
        "y": 180,
        "wires": [
            [
                "f47b807c9e2a40bf"
            ]
        ]
    },
    {
        "id": "e6cec9db5cd72f78",
        "type": "telegram bot",
        "botname": "ShoplifeRPA_bot",
        "usernames": "",
        "chatids": "",
        "baseapiurl": "",
        "testenvironment": false,
        "updatemode": "polling",
        "pollinterval": 300,
        "usesocks": false,
        "sockshost": "",
        "socksprotocol": "socks5",
        "socksport": 6667,
        "socksusername": "anonymous",
        "sockspassword": "",
        "bothost": "",
        "botpath": "",
        "localbothost": "0.0.0.0",
        "localbotport": 8443,
        "publicbotport": 8443,
        "privatekey": "",
        "certificate": "",
        "useselfsignedcertificate": false,
        "sslterminated": false,
        "verboselogging": false
    },
    {
        "id": "ed5cb154bf41a709",
        "type": "ui_group",
        "name": "RPA현황",
        "tab": "5ed6e18ff0aaa705",
        "order": 1,
        "disp": true,
        "width": 24,
        "collapse": false,
        "className": ""
    },
    {
        "id": "5ed6e18ff0aaa705",
        "type": "ui_tab",
        "name": "RPA 모니터링",
        "icon": "dashboard",
        "disabled": false,
        "hidden": false
    },
    {
        "id": "8c1ab3b844cfaddc",
        "type": "global-config",
        "env": [],
        "modules": {
            "node-red-contrib-syslog-input2": "1.0.4",
            "node-red-node-email": "3.1.0",
            "node-red-contrib-telegrambot": "17.0.3",
            "node-red-node-ui-table": "0.4.5",
            "node-red-dashboard": "3.6.6"
        }
    }
] ```

</details>


3. 📸 실제 구동 화면 (Demo)
(Node-REDflow.png)
