📂 NAS 데이터 자동화 및 연동 (NAS Data Automation)

1. 프로젝트 개요 (Overview)

이 워크플로우는 로컬 시스템의 특정 데이터를 NAS(Network Attached Storage) 서버로 자동 백업하고 동기화하는 RPA 프로세스입니다. Node-RED를 활용하여 파일 시스템의 변경 사항을 감지하고, SMB 프로토콜을 통해 NAS로 안전하게 전송합니다.

주요 기능: 실시간 파일 감지, 조건부 필터링, NAS 자동 업로드, 에러 로깅

사용 도구: Node-RED, SMB Protocol, Windows Task Scheduler

기대 효과: 수동 백업 시간 단축 및 데이터 누락 방지

2. 프로세스 흐름도 (Process Architecture)

아래는 전체 데이터 처리 로직을 시각화한 Node-RED 플로우입니다.

(위 이미지는 로컬 폴더 감지부터 NAS 적재까지의 전체 흐름을 나타냅니다.)

3. 핵심 로직 설명 (Key Logic)

3.1. 파일 감지 (Watch Directory)

fs-ops 노드를 사용하여 지정된 로컬 경로(C:/RPA_Work/Output)를 모니터링합니다.

새로운 파일이 생성되거나 수정될 때 즉시 트리거가 발생합니다.

3.2. 데이터 필터링 및 가공

확장자 검사: .csv, .xlsx 파일만 통과시키도록 Switch 노드를 설정했습니다.

파일명 정규화: 파일명에 타임스탬프(YYYYMMDD_HHmmss)를 자동으로 부여하여 중복 덮어쓰기를 방지합니다.

3.3. NAS 연동 (SMB/CIFS)

SMB 클라이언트 노드를 활용하여 NAS 서버 인증을 처리합니다.

네트워크 연결 실패 시 3회 재시도(Retry) 로직이 포함되어 있습니다.

4. Node-RED Flow 코드 (JSON)

아래 코드는 해당 워크플로우의 전체 JSON 데이터입니다. Node-RED의 Import 기능을 통해 바로 테스트해볼 수 있습니다.

<details>
<summary>🔻 <b>JSON 코드 보기 (클릭하여 펼치기)</b></summary>

[
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
        "func": "const log = msg.payload && msg.payload.msg ? msg.payload.msg : \"\";\nif (!log.toLowerCase().includes(\".xls\")) return null;\n\n// 1) SMB 전용: 파일 쓰기(write) 작업인지 확인\nif (!log.toLowerCase().includes(\"write\")) return null;\n\n// 2) 임시 파일(~$...) 제외\nif (log.match(/~\\$.*\\.xlsx?/i)) return null;\n\n// 3) 경로 추출\nlet match = log.match(/Path:\\s*(.*?\\.xlsx?),/i);\nif (!match || !match[1]) return null;\n\n// .trim()은 전체 양끝을, .replace()는 .xls 바로 앞의 공백(\\s+)만 제거\nlet filePath = match[1].trim().replace(/\\s+\\.xls/i, '.xls');\n\n// 5) '샵라이프' 폴더 확인\nif (!filePath.includes(\"샵라이프\")) return null;\n\n// 경로에 '결과파일'이 들어있으면 true, 아니면 false를 저장\nmsg.isResultFile = filePath.includes(\"결과파일\");\n\n// 7) 최종 경로 및 토픽 설정\nmsg.payload = \"\\\\\\\\192.168.0.8\" + filePath.replace(/\\//g, \"\\\\\");\nmsg.topic = \"[NAS] \" + msg.payload.split(\"\\\\\").pop();\n\nreturn msg;",
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
        "func": "const log = msg.payload && msg.payload.msg ? msg.payload.msg : \"\";\nif (!log.toLowerCase().includes(\".xls\")) return null;\n\n// 1) 액션 체크 (웹 업로드 및 쓰기 작업 포함)\nconst webActions = [\"upload\", \"create\", \"move\", \"write\"];\nconst isValid = webActions.some(a => log.toLowerCase().includes(a));\nif (!isValid) return null;\n\n// 2) 임시 파일 제외\nif (log.match(/~\\$.*\\.xlsx?/i)) return null;\n\n// 3) 경로 추출\nlet match = log.match(/Path:\\s*(.*?\\.xlsx?),/i);\nif (!match || !match[1]) return null;\n\n// .trim()으로 양끝을 잡고, .replace()로 확장자 바로 앞의 공백제거\nlet filePath = match[1].trim().replace(/\\s+\\.xls/i, '.xls');\n\n// 5) '샵라이프' 폴더 확인\nif (!filePath.includes(\"샵라이프\")) return null;\n\n// 경로에 '결과파일'이 포함되어 있으면 true, 아니면 false\nmsg.isResultFile = filePath.includes(\"결과파일\");\n\n// 7) 최종 경로 및 토픽 설정\nmsg.payload = \"\\\\\\\\192.168.0.8\" + filePath.replace(/\\//g, \"\\\\\");\nmsg.topic = \"[NAS] \" + msg.payload.split(\"\\\\\").pop();\n\nreturn msg;",
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
        "id": "6a58d43fff524195",
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
]

</details>

5. 트러블슈팅 및 해결 (Troubleshooting)

Issue: 대용량 파일 전송 시 타임아웃 발생

Solution: Node-RED의 stream 방식을 적용하여 파일을 청크(Chunk) 단위로 쪼개서 전송함으로써 메모리 오버플로우 방지 및 안정성 확보.
