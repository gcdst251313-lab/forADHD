# 점자 번역 플랫폼 소스코드 

아래는 프로젝트의 전체 소스코드입니다.

```<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>FocusFlow AI</title>
<style>
body{
    margin:0;
    font-family:Arial, sans-serif;
    background:linear-gradient(135deg,#74ebd5,#ACB6E5);
    min-height:100vh;
    display:flex;
    justify-content:center;
    align-items:center;
    padding:20px;
}
.container{
    width:100%;
    max-width:800px;
    background:white;
    border-radius:20px;
    padding:30px;
    box-shadow:0 10px 30px rgba(0,0,0,0.2);
}
h1{ text-align:center; color:#333; }
.subtitle{ text-align:center; color:#666; margin-bottom:30px; }
.card{
    background:#f7f9fc;
    border-radius:15px;
    padding:20px;
    margin-bottom:20px;
}
input{
    width:100%;
    padding:15px;
    border: 1px solid #ddd;
    border-radius:10px;
    font-size:16px;
    margin-bottom:15px;
    box-sizing:border-box;
}
button{
    padding:12px 20px;
    border:none;
    border-radius:10px;
    background:#4a90e2;
    color:white;
    cursor:pointer;
    font-size:16px;
    margin-right:10px;
    transition:0.3s;
}
button:hover{ background:#357bd8; }
button:disabled { background: #ccc; cursor: not-allowed; }
ul{ padding-left:20px; }
li{
    margin-bottom:10px;
    font-size:17px;
    display: flex;
    justify-content: space-between;
    align-items: center;
}
#timer{
    font-size:60px;
    text-align:center;
    margin:20px 0;
    color:#4a90e2;
    font-weight:bold;
}
.progress-bar{
    width:100%;
    height:20px;
    background:#ddd;
    border-radius:20px;
    overflow:hidden;
    margin-top:15px;
}
.progress{
    height:100%;
    width:0%;
    background:#4a90e2;
    transition:0.3s;
}
.message{
    font-size:20px;
    text-align:center;
    color:#444;
    min-height:40px;
    font-weight: bold;
}
.small{ color:#777; font-size:14px; margin-top:10px; }
.complete-btn{ background:#2ecc71; margin-left:10px; padding: 8px 15px;}
.complete-btn:hover{ background:#27ae60; }
.stats{ display:flex; justify-content:space-between; text-align:center; margin-top:15px; }
.stat-box{ flex:1; background:white; padding:15px; border-radius:10px; margin:5px; box-shadow:0 2px 5px rgba(0,0,0,0.05);}
.stat-number{ font-size:28px; color:#4a90e2; font-weight:bold; }
</style>
</head>
<body>

<div class="container">
    <h1>🧠 FocusFlow AI</h1>
    <p class="subtitle">ADHD 학생을 위한 집중 & 학습 보조 시스템</p>

    <div class="card">
        <h2>📌 오늘 해야 할 일</h2>
        <input type="text" id="taskInput" placeholder="예: 수학 숙제 하기, 방 청소하기" />
        <button id="analyzeBtn" onclick="analyzeTask()">AI 분석 시작</button>
        <div class="small">큰 과제를 작은 단계로 나누면 시작하기 쉬워집니다.</div>
    </div>

    <div class="card">
        <h2>🪄 AI가 만든 작은 단계</h2>
        <ul id="subtasks">
            <li style="color:#999; font-size:15px;">할 일을 입력하면 AI가 가장 쉬운 단계로 쪼개줍니다!</li>
        </ul>
        <div class="progress-bar">
            <div class="progress" id="progress"></div>
        </div>
    </div>

    <div class="card">
        <h2>⏰ 집중 타이머</h2>
        <div id="timer">25:00</div>
        <button onclick="startTimer()">집중 시작</button>
        <button onclick="pauseTimer()">일시정지</button>
        <button onclick="resetTimer()">초기화</button>
    </div>

    <div class="card">
        <h2>💬 AI 실시간 응원</h2>
        <div class="message" id="message">
            오늘도 작은 성공을 만들어봅시다 🚀
        </div>
    </div>

    <div class="card">
        <h2>📊 오늘의 진행 상황</h2>
        <div class="stats">
            <div class="stat-box">
                <div class="stat-number" id="completedCount">0</div>
                <div>완료한 단계</div>
            </div>
            <div class="stat-box">
                <div class="stat-number" id="focusSessions">0</div>
                <div>집중 세션</div>
            </div>
        </div>
    </div>
</div>

<script>
let completedTasks = 0;
let totalTasks = 0;
let focusCount = 0;

// API 키 없이 작동하는 무료 AI 연동 함수 (Pollinations.ai 활용)
async function fetchFreeAI(prompt, isJson = false) {
    try {
        const response = await fetch('https://text.pollinations.ai/', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                messages: [
                    {
                        role: 'system',
                        content: isJson ?
                            '당신은 작업을 작게 쪼개주는 AI입니다. 무조건 ["단계1", "단계2", "단계3"] 형태의 JSON 배열(Array)로만 대답하세요. 다른 말은 절대 금지.' :
                            '당신은 ADHD 학생을 응원하는 코치입니다. 한국어로 짧고 다정한 응원 한 문장만 작성하세요. 이모지 필수.'
                    },
                    { role: 'user', content: prompt }
                ],
                seed: Math.random() // 항상 새로운 답변을 받기 위한 난수
            })
        });

        if (!response.ok) throw new Error("API 응답 오류");
       
        let text = await response.text();

        if (isJson) {
            // AI가 마크다운을 포함할 경우를 대비하여 텍스트 정제
            text = text.replace(/```json/g, "").replace(/```/g, "").trim();
            return JSON.parse(text);
        }
        return text;

    } catch (error) {
        console.warn("AI 호출 중 에러 발생, 기본값으로 대체합니다:", error);
        // 인터넷 연결 문제나 무료 서버 에러 시 사이트가 멈추지 않도록 기본값 반환
        if (isJson) {
            return ["주변 1분 정리하기", "자리에 앉아서 심호흡하기", "딱 5분만 시작해보기", "나머지 부분 마무리하기"];
        } else {
            return "아주 잘하고 있어요! 조금만 더 힘내볼까요? 🚀";
        }
    }
}

// 할 일 분석
async function analyzeTask() {
    const input = document.getElementById("taskInput").value.trim();
    if (input === "") {
        alert("할 일을 입력해주세요!");
        return;
    }

    const btn = document.getElementById("analyzeBtn");
    const list = document.getElementById("subtasks");
    const msg = document.getElementById("message");

    // 로딩 상태 처리
    btn.innerText = "AI가 생각 중... ⏳";
    btn.disabled = true;
    list.innerHTML = "<li style='color:#4a90e2;'>로딩 중: 할 일을 가장 시작하기 쉬운 단계로 쪼개고 있어요...</li>";
    msg.innerText = "잠시만 기다려주세요!";

    // 1. 작은 단계로 쪼개는 프롬프트
    const taskPrompt = `학생이 '${input}'라는 할 일을 하려고 해. 5분 안에 끝낼 수 있는 아주 작은 행동 3~5개로 쪼개줘.`;
    const subtasks = await fetchFreeAI(taskPrompt, true);
   
    // 2. 시작 응원 메시지 프롬프트
    const msgPrompt = `학생이 방금 '${input}'를 하기 위해 계획을 세웠어. 바로 시작할 수 있도록 행동을 촉구하는 짧은 한 문장의 긍정적인 응원을 해줘.`;
    const encouragement = await fetchFreeAI(msgPrompt, false);
   
    // 화면 업데이트
    renderSubtasks(subtasks);
    msg.innerText = encouragement;
   
    // 버튼 복구
    btn.innerText = "AI 분석 시작";
    btn.disabled = false;
}

// 서브태스크 렌더링
function renderSubtasks(tasks) {
    const list = document.getElementById("subtasks");
    list.innerHTML = "";
    completedTasks = 0;
    totalTasks = tasks.length;
    updateProgress();

    tasks.forEach(task => {
        const li = document.createElement("li");
        const span = document.createElement("span");
        span.innerText = task;

        const btn = document.createElement("button");
        btn.innerText = "완료";
        btn.className = "complete-btn";

        btn.onclick = function() {
            if (!li.classList.contains("done")) {
                li.style.textDecoration = "line-through";
                li.style.color = "#999";
                li.classList.add("done");
               
                btn.style.display = "none";

                completedTasks++;
                document.getElementById("completedCount").innerText = completedTasks;
                updateProgress();

                generateEncouragement(task);
            }
        };

        li.appendChild(span);
        li.appendChild(btn);
        list.appendChild(li);
    });
}

// 작은 단계 완료 시 칭찬 메시지 생성
async function generateEncouragement(completedTaskName) {
    const msg = document.getElementById("message");
    msg.innerText = "칭찬 메시지 생성 중... ✨";

    let msgPrompt;
    if (completedTasks === totalTasks) {
         msgPrompt = `학생이 계획한 모든 단계를 전부 완료했어! 폭풍 칭찬과 성취감을 주는 짧고 감동적인 한 문장을 작성해줘.`;
    } else {
         msgPrompt = `학생이 방금 '${completedTaskName}' 단계를 완료했어! 아주 잘했다고 칭찬하면서 다음 단계로 갈 수 있게 동기부여하는 짧은 한 문장을 써줘.`;
    }
   
    const encouragement = await fetchFreeAI(msgPrompt, false);
    msg.innerText = encouragement;
}

// 프로그레스 바 업데이트
function updateProgress() {
    const percent = totalTasks === 0 ? 0 : (completedTasks / totalTasks) * 100;
    document.getElementById("progress").style.width = percent + "%";
}


// --- 타이머 관련 로직 ---
let timer;
let timeLeft = 1500; // 25분

function startTimer() {
    clearInterval(timer);
    timer = setInterval(() => {
        if (timeLeft <= 0) {
            clearInterval(timer);
            focusCount++;
            document.getElementById("focusSessions").innerText = focusCount;
            alert("집중 세션 완료! 🎉 잠시 휴식하세요.");
           
            // 타이머 완료 시 AI 응원 요청 (비동기)
            const prompt = "학생이 25분 집중 타이머를 끝냈어! 고생했다고 격려하고 5분간 휴식하라고 다정하게 한 문장으로 말해줘.";
            fetchFreeAI(prompt, false).then(res => {
                document.getElementById("message").innerText = res;
            });

            return;
        }
        timeLeft--;
        updateTimer();
    }, 1000);
}

function pauseTimer() { clearInterval(timer); }

function resetTimer() {
    clearInterval(timer);
    timeLeft = 1500;
    updateTimer();
}

function updateTimer() {
    const minutes = String(Math.floor(timeLeft / 60)).padStart(2, "0");
    const seconds = String(timeLeft % 60).padStart(2, "0");
    document.getElementById("timer").innerText = `${minutes}:${seconds}`;
}
</script>

</body>
</html>


```
