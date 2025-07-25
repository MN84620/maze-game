# maze-game
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Maze Game (3 Levels)</title>
  <style>
    body { font-family: sans-serif; text-align: center; background: #f5f5f5; }
    h1 { margin: 10px 0; }
    #ui { margin: 10px; }
    #timer, #level-info { font-size: 18px; font-weight: 600; margin: 5px; }
    button { margin: 5px; padding: 5px 10px; font-size: 16px; }
    canvas { background: white; border: 2px solid black; margin-top: 10px; touch-action: none; }
    #records { margin-top: 20px; text-align: left; display: inline-block; }
    table { border-collapse: collapse; margin: 0 auto; }
    td, th { border: 1px solid #aaa; padding: 5px 8px; }
  </style>
</head>
<body>
  <h1>Maze Game (3 Levels)</h1>
  <div id="ui">
    <button onclick="setDifficulty('easy')">ง่าย</button>
    <button onclick="setDifficulty('medium')">ปานกลาง</button>
    <button onclick="setDifficulty('hard')">ยาก</button>
    <button onclick="showRecords()">ดูบันทึกเวลา</button>
    <button onclick="resetGame()">รีเซ็ต</button>
  </div>
  <div id="level-info">เลือกความยากเพื่อเริ่มเกม</div>
  <div id="timer">เวลา: 0.00 วินาที</div>
  <canvas id="maze" width="480" height="480"></canvas>
  <div id="records"></div>

  <script>
    const canvas = document.getElementById('maze');
    const ctx = canvas.getContext('2d');
    const timerEl = document.getElementById('timer');
    const levelInfo = document.getElementById('level-info');
    const recordsEl = document.getElementById('records');

    const START = { x: 0, y: 0, w: 48, h: 48 };
    const GOAL  = { x: 432, y: 432, w: 48, h: 48 };

    let wallMap = null;
    let difficulty = null;
    let currentRound = 0;
    const roundsPerLevel = 3;
    let records = { easy: [], medium: [], hard: [] };

    let isDrawing = false;
    let finished = false;
    let startTime = 0;
    let interval = null;
    let lastX = 0, lastY = 0;

    // -----------------------------
    // Maze Patterns (10x10)
    // -----------------------------
    const mazePatterns = {
      easy: [
        "S_________",
        "#########_",
        "________#_",
        "_######__",
        "_#____#__",
        "_#_###_#G",
        "_#___###_",
        "___#_#___",
        "##_#_####",
        "__________"
      ],
      medium: [
        "S__#____#_",
        "##_#_##__",
        "________#_",
        "_######_##",
        "_#____#__",
        "_#_###_#G",
        "_#___###_",
        "___#_#___",
        "##_#_####",
        "__________"
      ],
      hard: [
        "S__#___#_",
        "##_#_##__",
        "________#_",
        "_######_##",
        "_#__#_#__",
        "_#_###_#G",
        "_#___###_",
        "___#_#___",
        "##_#_####",
        "__________"
      ]
    };

    function setDifficulty(level) {
      difficulty = level;
      currentRound = 0;
      recordsEl.innerHTML = '';
      drawMazeByDifficulty(level);
      levelInfo.textContent = `ระดับ: ${level} (รอบ ${currentRound + 1} / ${roundsPerLevel})`;
    }

    function showRecords() {
      let html = '<h2>บันทึกเวลา</h2><table><tr><th>ระดับ</th><th>รอบ</th><th>เวลา (วินาที)</th></tr>';
      for (const lv of ['easy', 'medium', 'hard']) {
        records[lv].forEach((time, i) => {
          html += `<tr><td>${lv}</td><td>${i+1}</td><td>${time.toFixed(2)}</td></tr>`;
        });
      }
      html += '</table>';
      recordsEl.innerHTML = html;
    }

    function drawMazeByDifficulty(level) {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = '#fff';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      const cell = 48;
      const maze = mazePatterns[level];
      ctx.fillStyle = '#000';

      for (let y = 0; y < maze.length; y++) {
        for (let x = 0; x < maze[y].length; x++) {
          if (maze[y][x] === "#") {
            ctx.fillRect(x * cell, y * cell, cell, cell);
          }
        }
      }

      ctx.fillStyle = 'green';
      ctx.fillRect(START.x, START.y, START.w, START.h);
      ctx.fillStyle = 'red';
      ctx.fillRect(GOAL.x, GOAL.y, GOAL.w, GOAL.h);

      wallMap = ctx.getImageData(0, 0, canvas.width, canvas.height);
      resetTimer();
    }

    function resetGame() {
      if (difficulty) {
        currentRound = 0;
        records[difficulty] = [];
        drawMazeByDifficulty(difficulty);
        levelInfo.textContent = `ระดับ: ${difficulty} (รอบ ${currentRound + 1} / ${roundsPerLevel})`;
      }
    }

    function resetTimer() {
      clearInterval(interval);
      timerEl.textContent = 'เวลา: 0.00 วินาที';
      finished = false;
      isDrawing = false;
    }

    function startTimer() {
      startTime = performance.now();
      interval = setInterval(() => {
        timerEl.textContent = 'เวลา: ' + ((performance.now() - startTime) / 1000).toFixed(2) + ' วินาที';
      }, 50);
    }

    function stopTimer() {
      clearInterval(interval);
    }

    function getElapsed() {
      return (performance.now() - startTime) / 1000;
    }

    function inRect(x, y, r) {
      return x >= r.x && x <= r.x + r.w && y >= r.y && y <= r.y + r.h;
    }

    function isWall(x, y) {
      if (x < 0 || y < 0 || x >= canvas.width || y >= canvas.height) return true;
      const idx = (y * canvas.width + x) * 4;
      const d = wallMap.data;
      return d[idx] === 0 && d[idx+1] === 0 && d[idx+2] === 0;
    }

    function pointerDown(x, y) {
      if (finished || !difficulty) return;
      if (inRect(x, y, START)) {
        isDrawing = true;
        ctx.beginPath();
        ctx.moveTo(x, y);
        lastX = x; lastY = y;
        startTimer();
      }
    }

    function pointerMove(x, y) {
      if (!isDrawing || finished) return;
      if (!isWall(x, y)) {
        ctx.lineTo(x, y);
        ctx.stroke();
        lastX = x; lastY = y;
      }
      if (inRect(lastX, lastY, GOAL)) {
        finished = true;
        isDrawing = false;
        stopTimer();
        const t = getElapsed();
        records[difficulty].push(t);
        alert(`ผ่านด่าน! เวลา: ${t.toFixed(2)} วินาที`);
        currentRound++;
        if (currentRound < roundsPerLevel) {
          drawMazeByDifficulty(difficulty);
          levelInfo.textContent = `ระดับ: ${difficulty} (รอบ ${currentRound + 1} / ${roundsPerLevel})`;
        } else {
          levelInfo.textContent = `ผ่านระดับ ${difficulty} ครบ ${roundsPerLevel} รอบ!`;
        }
      }
    }

    function pointerUp() { isDrawing = false; }

    function getPos(e) {
      const rect = canvas.getBoundingClientRect();
      if (e.touches && e.touches.length > 0) {
        const t = e.touches[0];
        return {
          x: Math.round((t.clientX - rect.left) * (canvas.width / rect.width)),
          y: Math.round((t.clientY - rect.top) * (canvas.height / rect.height))
        };
      } else {
        return {
          x: Math.round((e.clientX - rect.left) * (canvas.width / rect.width)),
          y: Math.round((e.clientY - rect.top) * (canvas.height / rect.height))
        };
      }
    }

    // Mouse
    canvas.addEventListener('mousedown', (e) => {
      const p = getPos(e);
      pointerDown(p.x, p.y);
    });
    canvas.addEventListener('mousemove', (e) => {
      const p = getPos(e);
      pointerMove(p.x, p.y);
    });
    canvas.addEventListener('mouseup', pointerUp);

    // Touch
    canvas.addEventListener('touchstart', (e) => {
      e.preventDefault();
      const p = getPos(e);
      pointerDown(p.x, p.y);
    }, { passive: false });
    canvas.addEventListener('touchmove', (e) => {
      e.preventDefault();
      const p = getPos(e);
      pointerMove(p.x, p.y);
    }, { passive: false });
    canvas.addEventListener('touchend', (e) => {
      e.preventDefault();
      pointerUp();
    }, { passive: false });
  </script>
</body>
</html>
