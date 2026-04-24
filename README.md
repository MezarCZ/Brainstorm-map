<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Brainstorming Neon - Cell Edit Edition</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #0a0a0b; color: white; 
            font-family: sans-serif; user-select: none;
        }
        #viewport { width: 100vw; height: 100vh; cursor: grab; position: relative; }
        #world { position: absolute; top: 0; left: 0; transform-origin: 0 0; }
        #line-canvas { position: absolute; top: 0; left: 0; pointer-events: none; }

        /* Bublina */
        .bubble { 
            padding: 15px; color: #ffffff;
            border: 2px solid rgba(255,255,255,0.1); border-radius: 16px; 
            position: absolute; cursor: move; min-width: 140px;
            box-shadow: 0 0 15px rgba(0,0,0,0.5); z-index: 2; outline: none;
            display: flex; flex-direction: column; align-items: center; text-align: center;
            backdrop-filter: blur(4px); animation: emerge 0.3s ease-out;
            font-weight: bold;
        }
        @keyframes emerge { 0% { transform: scale(0); } 100% { transform: scale(1); } }
        .bubble:focus { border-color: white !important; box-shadow: 0 0 25px rgba(255,255,255,0.5); }

        /* Mini Paleta uvnitř bubliny */
        .cell-picker {
            position: absolute; top: -35px; left: 50%; transform: translateX(-50%);
            display: none; gap: 5px; background: rgba(0,0,0,0.8); 
            padding: 5px; border-radius: 20px; border: 1px solid rgba(255,255,255,0.2);
            z-index: 20;
        }
        .bubble:focus-within .cell-picker { display: flex; }
        .cell-dot { width: 16px; height: 16px; border-radius: 50%; cursor: pointer; border: 1px solid rgba(255,255,255,0.3); }
        .cell-dot:hover { transform: scale(1.3); }

        .connector {
            width: 14px; height: 14px; background: white; border-radius: 50%;
            position: absolute; bottom: -7px; cursor: crosshair;
            opacity: 0; transition: opacity 0.2s; z-index: 10;
            box-shadow: 0 0 10px white;
        }
        .bubble:hover .connector { opacity: 1; }

        /* UI Ovládání */
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
        txt('hint-box', "Klikni na bublinu pro zm\u011Bnu barvy | T\u00E1hni z te\u010Dky pro ru\u010Dn\u00ED spojen\u00ED");

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
                        const force = (220 - dist) / 60;
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

        viewport.onmousedown = (e) => { if (e.target === viewport) { isDraggingView = true; startX = e.clientX - posX; startY = e.clientY - posY; } };
        window.onmousemove = (e) => {
            if (isDraggingView) { posX = e.clientX - startX; posY = e.clientY - startY; world.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`; }
            if (drawingLineFrom) tempMousePos = { x: (e.clientX - posX) / scale, y: (e.clientY - posY) / scale };
        };
        window.onmouseup = () => { isDraggingView = false; drawingLineFrom = null; tempMousePos = null; };

        function createBubbleElement(x, y, color) {
            const el = document.createElement('div');
            el.className = 'bubble';
            updateBubbleStyle(el, color);
            el.contentEditable = true;

            // Mini paleta
            const picker = document.createElement('div');
            picker.className = 'cell-picker';
            PALETTE.forEach(c => {
                const dot = document.createElement('div');
                dot.className = 'cell-dot';
                dot.style.background = c;
                dot.onmousedown = (e) => {
                    e.stopPropagation(); e.preventDefault();
                    const node = nodes.find(n => n.el === el);
                    node.color = c;
                    updateBubbleStyle(el, c);
                    saveToLocalStorage();
                };
                picker.appendChild(dot);
            });

            const conn = document.createElement('div');
            conn.className = 'connector';
            conn.onmousedown = (e) => { e.stopPropagation(); drawingLineFrom = nodes.find(n => n.el === el); };

            el.appendChild(picker);
            el.appendChild(conn);

            el.onmouseup = (e) => {
                if (drawingLineFrom && drawingLineFrom.el !== el) {
                    const target = nodes.find(n => n.el === el);
                    if (!target.manualParents.includes(drawingLineFrom)) { target.manualParents.push(drawingLineFrom); saveToLocalStorage(); }
                }
            };

            el.onmousedown = (e) => {
                if (e.target.className.includes('connector') || e.target.className.includes('cell-dot')) return;
                e.stopPropagation();
                if (e.button === 2) { deleteNode(el); return; }
                let bStartX = e.clientX / scale - x; let bStartY = e.clientY / scale - y;
                const move = (ev) => { const node = nodes.find(n => n.el === el); node.x = (ev.clientX / scale - bStartX); node.y = (ev.clientY / scale - bStartY); };
                document.addEventListener('mousemove', move);
                document.onmouseup = () => document.removeEventListener('mousemove', move);
            };
            el.oncontextmenu = (e) => e.preventDefault();
            el.oninput = () => saveToLocalStorage();

            world.appendChild(el);
            return el;
        }

        function updateBubbleStyle(el, color) {
            el.style.background = color + "CC";
            el.style.borderColor = color;
            el.style.boxShadow = `0 0 20px ${color}66`;
        }

        function deleteNode(el) {
            el.remove();
            nodes = nodes.filter(n => n.el !== el);
            nodes.forEach(n => n.manualParents = n.manualParents.filter(p => p.el !== el));
            saveToLocalStorage();
        }

        function addNode() {
            const x = (window.innerWidth / 2 - posX) / scale - 70;
            const y = (window.innerHeight / 2 - posY) / scale - 25;
            const el = createBubbleElement(x, y, currentColor);
            const sameColor = nodes.filter(n => n.color === currentColor);
            const autoParent = sameColor.length > 0 ? sameColor[sameColor.length - 1] : null;
            nodes.push({ el, x, y, color: currentColor, autoParent, manualParents: [] });
            setTimeout(() => el.focus(), 10);
            saveToLocalStorage();
        }

        function drawLines() {
            canvas.width = 10000; canvas.height = 10000;
            canvas.style.left = "-5000px"; canvas.style.top = "-5000px";
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.lineWidth = 4 / scale;

            nodes.forEach(node => {
                if (node.autoParent) drawCurve(node.autoParent.x + node.autoParent.el.offsetWidth/2 + 5000, node.autoParent.y + node.autoParent.el.offsetHeight/2 + 5000, node.x + node.el.offsetWidth/2 + 5000, node.y + node.el.offsetHeight/2 + 5000, node.autoParent.color, node.color);
                node.manualParents.forEach(p => drawCurve(p.x + p.el.offsetWidth/2 + 5000, p.y + p.el.offsetHeight/2 + 5000, node.x + node.el.offsetWidth/2 + 5000, node.y + node.el.offsetHeight/2 + 5000, p.color, node.color));
            });

            if (drawingLineFrom && tempMousePos) drawCurve(drawingLineFrom.x + drawingLineFrom.el.offsetWidth/2 + 5000, drawingLineFrom.y + drawingLineFrom.el.offsetHeight/2 + 5000, tempMousePos.x + 5000, tempMousePos.y + 5000, drawingLineFrom.color, "#ffffff");
        }

        function drawCurve(x1, y1, x2, y2, c1, c2) {
            const grad = ctx.createLinearGradient(x1, y1, x2, y2);
            grad.addColorStop(0, c1); grad.addColorStop(1, c2);
            ctx.strokeStyle = grad; ctx.shadowBlur = 15 / scale; ctx.shadowColor = c1;
            ctx.beginPath(); ctx.moveTo(x1, y1);
            const cp1x = x1 + (x2 - x1) * 0.5;
            ctx.bezierCurveTo(cp1x, y1, cp1x, y2, x2, y2);
            ctx.stroke(); ctx.shadowBlur = 0;
        }

        function saveToLocalStorage() {
            const data = nodes.map(n => ({ x: n.x, y: n.y, text: n.el.innerText, color: n.color, autoParentIdx: nodes.indexOf(n.autoParent), manualParentIndices: n.manualParents.map(p => nodes.indexOf(p)) }));
            localStorage.setItem('myFinalNeonMap', JSON.stringify(data));
        }

        function exportToFile() { saveToLocalStorage(); const data = localStorage.getItem('myFinalNeonMap'); const blob = new Blob([data], { type: 'application/json' }); const url = URL.createObjectURL(blob); const a = document.createElement('a'); a.href = url; a.download = 'projekt.json'; a.click(); }
        function importFromFile(event) { const file = event.target.files[0]; if (!file) return; const reader = new FileReader(); reader.onload = (e) => loadFromData(JSON.parse(e.target.result)); reader.readAsText(file); }

        function loadFromData(data) {
            nodes.forEach(n => n.el.remove()); nodes = [];
            data.forEach(d => { const el = createBubbleElement(d.x, d.y, d.color); el.innerText = d.text; nodes.push({ el, x: d.x, y: d.y, color: d.color, autoParent: null, manualParents: [] }); });
            data.forEach((d, i) => { if (d.autoParentIdx !== -1) nodes[i].autoParent = nodes[d.autoParentIdx]; d.manualParentIndices.forEach(idx => nodes[i].manualParents.push(nodes[idx])); });
        }

        window.onload = () => { const saved = localStorage.getItem('myFinalNeonMap'); if (saved) loadFromData(JSON.parse(saved)); applyPhysics(); };
    </script>
</body>
</html>
