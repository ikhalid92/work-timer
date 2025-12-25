<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WorkFlow Tracker</title>
    <style>
        :root {
            --bg: #f0f2f5;
            --card: #ffffff;
            --text: #1c1e21;
            --primary: #007bff;
            --secondary: #65676b;
            --accent: #00c853;
            --break: #ff9800;
        }

        body {
            font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg);
            color: var(--text);
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 20px;
        }

        .container {
            width: 100%;
            max-width: 450px;
        }

        .main-card {
            background: var(--card);
            border-radius: 24px;
            padding: 24px;
            box-shadow: 0 8px 30px rgba(0,0,0,0.08);
            margin-bottom: 20px;
        }

        .header { text-align: center; margin-bottom: 20px; }
        .header h1 { font-size: 1.5rem; margin: 0; color: var(--primary); }

        .time-display {
            text-align: center;
            padding: 20px;
            background: #f8f9fa;
            border-radius: 20px;
            margin-bottom: 20px;
        }

        .label { font-size: 0.8rem; color: var(--secondary); text-transform: uppercase; letter-spacing: 1px; }
        .exit-time { font-size: 2.5rem; font-weight: 800; color: var(--accent); display: block; }

        .stats-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 20px; }
        .stat-item { background: #f8f9fa; padding: 12px; border-radius: 12px; text-align: center; }
        .stat-val { font-weight: 700; display: block; }

        button {
            width: 100%;
            padding: 16px;
            border: none;
            border-radius: 14px;
            font-size: 1rem;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.2s;
            margin-bottom: 12px;
        }

        .btn-start { background: var(--primary); color: white; }
        .btn-break { background: var(--break); color: white; }
        .btn-finish { background: var(--text); color: white; }
        
        .history-card {
            background: var(--card);
            border-radius: 20px;
            padding: 20px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.05);
        }

        .history-item {
            display: flex;
            justify-content: space-between;
            padding: 10px 0;
            border-bottom: 1px solid #eee;
            font-size: 0.9rem;
        }

        .hidden { display: none; }
    </style>
</head>
<body>

<div class="container">
    <div class="main-card">
        <div class="header">
            <h1>Daily Work Tracker</h1>
        </div>

        <div class="time-display">
            <span class="label">Leave Today At</span>
            <span id="exitTime" class="exit-time">--:--</span>
        </div>

        <div class="stats-grid">
            <div class="stat-item">
                <span class="label">Started</span>
                <span id="startVal" class="stat-val">--:--</span>
            </div>
            <div class="stat-item">
                <span class="label">Breaks</span>
                <span id="breakVal" class="stat-val">0 min</span>
            </div>
        </div>

        <button id="btnIn" class="btn-start" onclick="handleIn()">Check In</button>
        <button id="btnOut" class="btn-break hidden" onclick="handleBreak()">Cafeteria / Break Out</button>
        <button id="btnResume" class="btn-start hidden" onclick="handleResume()">Back to Work</button>
        <button id="btnFinish" class="btn-finish hidden" onclick="handleFinish()">Finish & Save Day</button>
    </div>

    <div class="history-card">
        <h3 style="margin-top:0">History</h3>
        <div id="historyList"></div>
        <button onclick="clearHistory()" style="background:none; color:red; font-size:0.7rem; padding:5px;">Clear History</button>
    </div>
</div>

<script>
    let state = JSON.parse(localStorage.getItem('workAppState')) || {
        start: null,
        breaks: 0,
        breakStart: null,
        mode: 'OFF', // OFF, WORKING, BREAK
        history: []
    };

    function updateUI() {
        const { start, breaks, mode, history } = state;
        
        // Update Stats
        document.getElementById('startVal').innerText = start ? formatTime(new Date(start)) : '--:--';
        document.getElementById('breakVal').innerText = breaks + ' min';

        // Calculate Exit Time
        if (start) {
            const exit = new Date(new Date(start).getTime() + (8 * 60 * 60 * 1000) + (breaks * 60 * 1000));
            document.getElementById('exitTime').innerText = formatTime(exit);
        }

        // Toggle Buttons
        document.getElementById('btnIn').classList.toggle('hidden', mode !== 'OFF');
        document.getElementById('btnOut').classList.toggle('hidden', mode !== 'WORKING');
        document.getElementById('btnResume').classList.toggle('hidden', mode !== 'BREAK');
        document.getElementById('btnFinish').classList.toggle('hidden', mode !== 'WORKING');

        // Render History
        const histList = document.getElementById('historyList');
        histList.innerHTML = history.map(day => `
            <div class="history-item">
                <span><b>${day.date}</b></span>
                <span>${day.start} - ${day.end} (${day.breaks}m break)</span>
            </div>
        `).reverse().join('');

        localStorage.setItem('workAppState', JSON.stringify(state));
    }

    function handleIn() {
        state.start = new Date();
        state.mode = 'WORKING';
        updateUI();
    }

    function handleBreak() {
        state.breakStart = new Date();
        state.mode = 'BREAK';
        updateUI();
    }

    function handleResume() {
        const diff = Math.round((new Date() - new Date(state.breakStart)) / 60000);
        state.breaks += diff;
        state.breakStart = null;
        state.mode = 'WORKING';
        updateUI();
    }

    function handleFinish() {
        if (!confirm("Finish work for today?")) return;
        
        const exit = document.getElementById('exitTime').innerText;
        const today = new Date().toLocaleDateString();
        
        state.history.push({
            date: today,
            start: formatTime(new Date(state.start)),
            end: exit,
            breaks: state.breaks
        });

        // Reset for next day
        state.start = null;
        state.breaks = 0;
        state.mode = 'OFF';
        updateUI();
    }

    function formatTime(date) {
        return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    }

    function clearHistory() { if(confirm("Clear history?")) { state.history = []; updateUI(); } }

    updateUI();
</script>

</body>
</html>
