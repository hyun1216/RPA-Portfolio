# 📂 NAS 연동 자동화 워크플로우 (NAS Integration Workflow)

## 1. 프로젝트 개요 (Overview)
이 서브페이지는 RPA 프로세스 내에서 **NAS(Network Attached Storage)** 와의 데이터 연동을 자동화한 워크플로우에 대한 기술서입니다. Node-RED를 활용하여 데이터 흐름을 제어하고, NAS 상의 파일을 효율적으로 관리/처리하는 것을 목적으로 합니다.

---

## 2. 워크플로우 로직 (Workflow Logic)
Node-RED를 통해 구성된 전체 데이터 처리 흐름입니다.

![Node-RED Workflow](./Node-REDflow.png)
> *위 이미지는 Node-RED에서 설계한 Flow의 캡처본입니다.*

---

## 3. 주요 기능 및 코드 설명 (Key Features & Code)

### 3.1 NAS 연동 및 데이터 처리
NAS 서버와의 통신 및 파일 처리를 담당하는 핵심 로직입니다.

**[주요 코드 설명]**
- **기능:** NAS 디렉토리 내 신규 파일 감지 (xlsx 형식의 파일만 작동하도록 설정)
- **구현 방식:** 주기적인 폴링(Polling) 방식 사용

```javascript
[
    {
        "id": "442f946d0e7d1584",
        "type": "function",
        "z": "a1f80dcd91645bf8",
        "name": "웹 전용 필터링",
        "func": "const log = msg.payload && msg.payload.msg ? msg.payload.msg : \"\";\nif (!log.toLowerCase().includes(\".xls\")) return null;\n\n// 1) 액션 체크 (웹 업로드 및 쓰기 작업 포함)\nconst webActions = [\"upload\", \"create\", \"move\", \"write\"];\nconst isValid = webActions.some(a => log.toLowerCase().includes(a));\nif (!isValid) return null;\n\n// 2) 임시 파일 제외\nif (log.match(/~\\$.*\\.xlsx?/i)) return null;\n\n// 3) 경로 추출\nlet match = log.match(/Path:\\s*(.*?\\.xlsx?),/i);\nif (!match || !match[1]) return null;\n\n// .trim()으로 양끝을 잡고, .replace()로 확장자 바로 앞의 공백제거\nlet filePath = match[1].trim().replace(/\\s+\\.xls/i, '.xls');\n\n// 5) '보안' 폴더 확인\nif (!filePath.includes(\"보안\")) return null;\n\n// 경로에 '결과파일'이 포함되어 있으면 true, 아니면 false\nmsg.isResultFile = filePath.includes(\"결과파일\");\n\n// 7) 최종 경로 및 토픽 설정\nmsg.payload = \"\\\\\\\\192.168.0.8\" + filePath.replace(/\\//g, \"\\\\\");\nmsg.topic = \"[NAS] \" + msg.payload.split(\"\\\\\").pop();\n\nreturn msg;",
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
    }
]
};
return msg;
```

### 3.2 데이터 파싱 및 가공
가져온 데이터를 RPA 봇이 처리할 수 있는 형태로 변환하는 과정입니다.

```javascript

var fullPath = msg.payload; 
var pathParts = fullPath.split(/[/\\]/); 
var managerName = pathParts[pathParts.length - 2]; 
var regex = /([A-Za-z]{2}-\d{5})/; 
var match = fullPath.match(regex);
var projectCode = match ? match[0] : "코드없음";
var now = new Date().toLocaleString();

msg.payload = {
    "담당자": managerName,
    "프로젝트코드": projectCode,
    "파일명": pathParts[pathParts.length - 1],
    "시간": now
};
msg.managerName = managerName;
msg.projectCode = projectCode;
msg.fullPath = fullPath;

return msg;

```

## 4. 데이터 파일 메일 & 텔레그램 & 대시보드 전송
추출 된 데이터 값을 RPA작업자가 쉽게 확인 할 수 있도록 데이터 전송 (대시보드코드)
```javascript

var logList = flow.get("rpaLogList") || [];
var now = new Date();
now.setHours(now.getHours() + 9); 
var timeString = now.getFullYear() + "." + (now.getMonth()+1) + "." + now.getDate() + ". " + now.getHours() + ":" + now.getMinutes();

logList.unshift({
    "시간": timeString,
    "담당자": msg.managerName,
    "코드": msg.projectCode,
    "경로": msg.fullPath
});
if (logList.length > 30) logList.pop();
flow.set("rpaLogList", logList);
msg.payload = logList;
return msg;

```
