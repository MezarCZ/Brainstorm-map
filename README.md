<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Brainstorming Neon - Stable Edit</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #0a0a0b; color: white; 
            font-family: sans-serif; user-select: none;
        }
        #viewport { width: 100vw; height: 100vh; cursor: grab; position: relative; }
        #world { position: absolute; top: 0; left: 0; transform-origin: 0 0; }
        #line-canvas { position: absolute; top: 0; left: 0; pointer-events: none; }

        .bubble { 
            padding: 15px; color: #ffffff;
            border: 2px solid rgba(255,255,255,0.1); border-radius: 16px; 
            position: absolute; cursor: move; min-width: 140px; min-height: 40px;
            box-shadow: 0 0 15px rgba(0,0,0,0.5); z-index: 2; outline: none;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            text-align: center; backdrop-filter: blur(4px);
            font-weight: bold; transition: box-shadow 0.3s, border-color 0.3s;
        }
        .bubble:focus { border-color: white !important; cursor: text; user-select: text; }

        .cell-picker {
            position: absolute; top: -40px; left: 50%; transform: translateX(-50%);
            display: none; gap: 8px; background: rgba(0,0,0,0.9); 
            padding: 6px; border-radius: 20px; border: 1px solid rgba(255,255,255,0.2);
            z-index: 20; box-shadow: 0 4px 15px rgba(0,0,0,0.5);
        }
        .bubble:focus-within .cell-picker { display: flex; }
        .cell-dot { width: 18px; height: 18px; border-radius: 50%; cursor: pointer; border: 1px solid rgba(255,255,255,0.3); transition: 0.2s; }
        .cell-dot:hover { transform: scale(1.3); border-color: white; }

        .connector {
            width: 14px; height: 14px; background: white; border-radius: 50%;
            position: absolute; bottom: -7px; cursor: crosshair;
            opacity: 0; transition: opacity 0.2s; z-index: 10;
            box-shadow: 0 0 10px white;
        }
        .bubble:hover .connector { opacity: 1; }

        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; flex-direction: column; gap: 10px; }
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; }
        .color-picker { display: flex; gap: 8px; background: rgba(255,255,255,0.05); padding: 8px; border-radius: 12px; border: 1px solid rgba(255,255,255,0.1); }
        .color-dot { width: 28px; height: 28px; border-radius: 50%; cursor: pointer; border: 2px solid transparent; transition: 0.2s; }
        .color-dot.active { border-color: white; transform: scale(1.2); }

        button { padding: 12px 18px; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; color: white; transition: 0.2s; }
        .btn-add { background: #1a73e8; }
        .btn-save { background: #1e8e3e; }
        .btn-load { background: #007bff; }
        .btn-img { background: #8e24aa; }

        .hint { position: fixed; bottom: 20px; left: 20px; color: #888; font-size: 0.85em; background: rgba(0,0,0,0.7); padding: 10px 15px; border-radius: 8px; }
    </style>
</head>
<body>

    <div class="controls-left">
        <button class="btn-add" id="addBtn" onclick="addNode()">...</button>
        <div class="color-picker" id="picker">
            <div class="color-dot active" style="background: #2a2a2a;" onclick="selectColor('#2a2a2a', this)"></div>
            <div class="color-dot" style="background: #ff4757;" onclick="selectColor('#ff4757', this)"></div>
            <div class="color-dot" style="background: #2ed573;" onclick="selectColor('#2ed573', this)"></div>
            <div class="color-dot" style="background: #1e90ff;" onclick="selectColor('#1e90ff', this)"></div>
            <div class="color-dot" style="background: #ffa502;" onclick="selectColor('#ffa502', this)"></div>
        </div>
    </div>

    <div class="controls-right">
        <button class="btn-img" id="imgBtn" onclick="exportToImage()">PNG</button>
        <button class="btn-save" id="saveBtn" onclick="exportToFile()">...</button>
        <button class="btn-load" id="loadBtn" onclick="document.getElementById('fileInput').click()">...</button>
        <input type="file" id="fileInput" style="display:none" onchange="importFromFile(event)">
    </div>

    <div id="hint-box" class="hint">...</div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
        </div>
    </div>

    <script>
        const txt = (id, s) => document.getElementById(id).textContent = s;
        txt('addBtn', "Nov\u00E1 my\u0161lenka");
        txt('saveBtn', "Ulo\u017Eit");
        txt('loadBtn', "Otev\u0159\u00EDt");
        txt('hint-box', "Klikni pro psan\u00ED | T\u00E1hni pro pohyb | Prav\u00E9 tla\u010D\u00EDtko: Smazat");

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        let nodes = [];
        let currentColor = '#2a2a2a';
        let scale = 1, posX = 0, posY = 0;
        let isDraggingView = false, startX, startY, drawingLineFrom = null, tempMousePos = null;

        const PALETTE = ['#2a2a2a', '#ff4757', '#2ed573', '#1e90ff', '#ffa502'];

        function applyPhysics() {
            nodes.forEach((n1, i) => {
                nodes.forEach((n2, j) => {
                    if (i === j) return;
                    const dx = n2.x - n1.x; const dy = n2.y - n1.y;
                    const dist = Math.sqrt(dx * dx + dy * dy);
                    if (dist < 220) {
                        const force = (220 - dist) / 100;
                        n1.x -= (dx / dist) * force; n1.y -= (dy / dist) * force;
                        n2.x += (dx / dist) * force; n2.y += (dy / dist) * force;
                    }
                });
                n1.el.style.left = n1.x + 'px'; n1.el.style.top = n1.y + 'px';
            });
            drawLines();
            requestAnimationFrame(applyPhysics);
        }

        function selectColor(color, el) {
            currentColor = color;
            document.querySelectorAll('.color-dot').forEach(d => d.classList.remove('active'));
            el.classList.add('active');
        }

        viewport.onmousedown =
