<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Brainstorming Neon - Ultimate Fixed</title>
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
            min-width: 140px; min-height: 50px;
            color: #ffffff;
            border: 2px solid rgba(255,255,255,0.1); border-radius: 16px; 
            position: absolute; cursor: move; 
            box-shadow: 0 0 15px rgba(0,0,0,0.5); z-index: 2; outline: none;
            display: flex; align-items: center; justify-content: center;
            backdrop-filter: blur(4px);
            font-weight: bold; transition: box-shadow 0.3s, border-color 0.3s;
        }

        .bubble-content {
            width: 100%; height: 100%;
            display: flex; align-items: center; justify-content: center;
            padding: 10px; box-sizing: border-box;
            outline: none; pointer-events: none;
        }
        
        .bubble[data-editing="true"] { border-color: white !important; box-shadow: 0 0 25px rgba(255,255,255,0.5); }
        .bubble[data-editing="true"] .bubble-content { 
            pointer-events: auto; cursor: text; user-select: text;
        }

        .cell-picker {
            position: absolute; top: -45px; left: 50%; transform: translateX(-50%);
            display: none; gap: 8px; background: rgba(0,0,0,0.9); 
            padding: 6px; border-radius: 20px; border: 1px solid rgba(255,255,255,0.2);
            z-index: 20; pointer-events: auto;
        }
        .bubble:hover .cell-picker, .bubble:focus-within .cell-picker { display: flex; }
        .cell-dot { width: 18px; height: 18px; border-radius: 50%; cursor: pointer; border: 1px solid rgba(255,255,255,0.3); }

        .connector {
            width: 14px; height: 14px; background: white; border-radius: 50%;
            position: absolute; bottom: -7px; left: calc(50% - 7px); cursor: crosshair;
            opacity: 0; transition: opacity 0.2s; z-index: 10; pointer-events: auto;
            box-shadow: 0 0 8px white;
        }
        .bubble:hover .connector { opacity: 1; }

        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; flex-direction: column; gap: 10px; }
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; }
        
        button { padding: 12px 18px; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; color: white; transition: 0.2s; }
        button:hover { transform: translateY(-2px); }
        .btn-add { background: #1a73e8; }
        .btn-save { background: #1e8e3e; }
        .btn-load { background: #007bff; }

        .hint { position: fixed; bottom: 20px; left: 20px; color: #888; font-size: 0.85em; background: rgba(0,0,0,0.7); padding: 10px 15px; border-radius: 8px; }
    </style>
</head>
<body>

    <div class="controls-left">
        <button class="btn-add" id="addBtn">Add</button>
        <div id="picker-container" style="display: flex; gap: 8px; background: rgba(255,255,255,0.05); padding: 8px; border-radius: 12px;"></div>
    </div>

    <div class="controls-right">
        <button class="btn-save" id="saveBtn">Save</button>
        <button id="loadBtn" style="background: #007bff;">Open</button>
        <input type="file" id="fileInput" style="display:none" onchange="importFromFile(event)">
    </div>

    <div id="hintBox" class="hint"></div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
        </div>
    </div>

    <script>
        // Neprůstřelná čeština přes kódování
        const cz = (s) => decodeURIComponent(escape(s));
        document.getElementById('addBtn').textContent = cz("Nov\xE1 my\u0161lenka");
        document.getElementById('saveBtn').textContent = cz("Ulo\u017Eit");
        document.getElementById('loadBtn').textContent = cz("Otev\u0159\u00EDt");
        document.getElementById('hintBox').textContent = cz("Plocha: Tah pro posun | Bublina: Tah pro pohyb, Dvojklik pro psan\xED");

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        let nodes = [];
        let currentColor = '#ff4757';
        let posX = 0, posY = 0, scale = 1;
        let isDraggingView = false, startX, startY, drawingLineFrom = null, tempMousePos = null;

        const PALETTE = ['#2a2a2a', '#ff4757', '#2ed573', '#1e90ff', '#ffa502'];
        const pickerCont = document.getElementById('picker-container');
        PALETTE.forEach(c => {
            const d = document.createElement('div');
            d.style.width = '28px'; d.style.height = '28px'; d.style.borderRadius = '50%';
            d.style.background = c; d.style.cursor = 'pointer'; d.style.border = '2px solid transparent';
            if(c === currentColor) d.style.borderColor = 'white';
            d.onclick = () => {
                currentColor = c;
                Array.from(pickerCont.children).forEach(child => child.style.borderColor = 'transparent');
                d.style.borderColor = 'white';
            };
            pickerCont.appendChild(d);
        });

        function applyPhysics() {
            nodes.forEach((n1, i) => {
                nodes.forEach((n2, j) => {
                    if (i === j) return;
                    const dx = n2.x - n1.x; const dy = n2.y - n1.y;
                    const dist = Math.sqrt(dx * dx + dy * dy);
                    if (dist < 180) {
                        const force = (180 - dist) / 60;
                        n1.x -= (dx / dist) * force; n1.y -= (dy / dist) * force;
                        n2.x += (dx / dist) * force; n2.y += (dy / dist) * force;
                    }
                });
                n1.el.style.left = n1.x + 'px'; n1.el.style.top = n1.y + 'px';
            });
            drawLines();
            requestAnimationFrame(applyPhysics);
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

            const content = document.createElement('div');
            content.className = 'bubble-content';
            content.contentEditable = "false";
            el.appendChild(content);

            const picker = document.createElement('div');
            picker.className = 'cell-picker';
            PALETTE.forEach(c => {
                const dot = document.createElement('div');
                dot.className = 'cell-dot';
                dot.style.background = c;
                dot.onclick = (e) => {
                    e.stopPropagation();
                    const node = nodes.find(n => n.el === el);
                    node.color = c;
                    updateBubbleStyle(el, c);
                    saveToLocalStorage();
                };
                picker.appendChild(dot);
            });

            const conn = document.createElement('div');
            conn.className = 'connector';
            conn.onmousedown = (e) => { e.stopPropagation(); e.preventDefault(); drawingLineFrom = nodes.find(n => n.el === el); };
            el.appendChild(picker); el.appendChild(conn);

            el.ondblclick = (e) => {
                e.stopPropagation();
                el.setAttribute('data-editing', 'true');
                content.contentEditable = "true";
                content.focus();
            };

            content.onblur = () => { el.setAttribute('data-editing', 'false'); content.contentEditable = "false"; saveToLocalStorage(); };

            el.onmousedown = (e) => {
                if (el.getAttribute('data-editing') === "true") return;
                if (e.button === 2) { deleteNode(el); return; }
                e.stopPropagation();
                let bStartX = e.clientX / scale - x;
                let bStartY = e.clientY / scale - y;
                const move = (ev) => { 
                    const node = nodes.find(n => n.el === el); 
                    node.x = (ev.clientX / scale - bStartX); node.y = (ev.clientY / scale - bStartY); 
                };
                document.addEventListener('mousemove', move);
                document.onmouseup = () => document.removeEventListener('mousemove', move);
            };

            el.onmouseup = (e) => {
                if (drawingLineFrom && drawingLineFrom.el !== el) {
                    const target = nodes.find(n => n.el === el);
                    if (!target.manualParents.includes(drawingLineFrom)) { target.manualParents.push(drawingLineFrom); saveToLocalStorage(); }
                }
            };
            
            el.oncontextmenu = (e) => e.preventDefault();
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
            saveToLocalStorage();
        }

        function drawLines() {
            canvas.width = 10000; canvas.height = 10000;
            canvas.style.left = "-5000px"; canvas.style.top = "-5000px";
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.lineWidth = 4;
            nodes.forEach(node => {
                if (node.autoParent) drawCurve(node.autoParent.x + node.autoParent.el.offsetWidth/2 + 5000, node.autoParent.y + node.autoParent.el.offsetHeight/2 + 5000, node.x + node.el.offsetWidth/2 + 5000, node.y + node.el.offsetHeight/2 + 5000, node.autoParent.color, node.color);
                node.manualParents.forEach(p => drawCurve(p.x + p.el.offsetWidth/2 + 5000, p.y + p.el.offsetHeight/2 + 5000, node.x + node.el.offsetWidth/2 + 5000, node.y + node.el.offsetHeight/2 + 5000, p.color, node.color));
            });
            if (drawingLineFrom && tempMousePos) drawCurve(drawingLineFrom.x + drawingLineFrom.el.offsetWidth/2 + 5000, drawingLineFrom.y + drawingLineFrom.el.offsetHeight/2 + 5000, tempMousePos.x + 5000, tempMousePos.y + 5000, drawingLineFrom.color, "#ffffff");
        }

        function drawCurve(x1, y1, x2, y2, c1, c2) {
            const grad = ctx.createLinearGradient(x1, y1, x2, y2);
            grad.addColorStop(0, c1); grad.addColorStop(1, c2);
            ctx.strokeStyle = grad; ctx.shadowBlur = 15; ctx.shadowColor = c1;
            ctx.beginPath(); ctx.moveTo(x1, y1);
            ctx.bezierCurveTo(x1 + (x2 - x1) * 0.5, y1, x1 + (x2 - x1) * 0.5, y2, x2, y2);
            ctx.stroke(); ctx.shadowBlur = 0;
        }

        function saveToLocalStorage() {
            const data = nodes.map(n => ({ x: n.x, y: n.y, text: n.el.querySelector('.bubble-content').innerText, color: n.color, autoParentIdx: nodes.indexOf(n.autoParent), manualParentIndices: n.manualParents.map(p => nodes.indexOf(p)) }));
            localStorage.setItem('myNeonHybridFinal', JSON.stringify(data));
        }

        function exportToFile() { saveToLocalStorage(); const data = localStorage.getItem('myNeonHybridFinal'); const blob = new Blob([data], { type: 'application/json' }); const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'projekt.json'; a.click(); }
        function importFromFile(event) { const file = event.target.files[0]; if (!file) return; const reader = new FileReader(); reader.onload = (e) => loadFromData(JSON.parse(e.target.result)); reader.readAsText(file); }

        function loadFromData(data) {
            nodes.forEach(n => n.el.remove()); nodes = [];
            data.forEach(d => { const el = createBubbleElement(d.x, d.y, d.color); el.querySelector('.bubble-content').innerText = d.text; nodes.push({ el, x: d.x, y: d.y, color: d.color, autoParent: null, manualParents: [] }); });
            data.forEach((d, i) => { if (d.autoParentIdx !== -1) nodes[i].autoParent = nodes[d.autoParentIdx]; d.manualParentIndices.forEach(idx => nodes[i].manualParents.push(nodes[idx])); });
        }

        window.onload = () => { const saved = localStorage.getItem('myNeonHybridFinal'); if (saved) loadFromData(JSON.parse(saved)); applyPhysics(); };
    </script>
</body>
</html>
