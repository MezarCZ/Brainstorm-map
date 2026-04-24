<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Brainstorm Neon - Designer Edition</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #0a0a0b; color: white; 
            font-family: 'Inter', sans-serif; user-select: none;
            /* JEMNÁ MŘÍŽKA NA POZADÍ */
            background-image: radial-gradient(rgba(255, 255, 255, 0.05) 1px, transparent 1px);
            background-size: 30px 30px;
        }
        #viewport { width: 100vw; height: 100vh; cursor: grab; position: relative; }
        #world { position: absolute; top: 0; left: 0; transform-origin: 0 0; }
        #line-canvas { position: absolute; top: 0; left: 0; pointer-events: none; }

        .bubble { 
            padding: 15px; color: #ffffff;
            border: 2px solid rgba(255,255,255,0.1); border-radius: 16px; 
            position: absolute; cursor: move; min-width: 140px;
            box-shadow: 0 0 15px rgba(0,0,0,0.5); z-index: 2; outline: none;
            display: flex; flex-direction: column; align-items: center; text-align: center;
            backdrop-filter: blur(8px); transition: box-shadow 0.3s, border-color 0.3s;
            font-weight: 500; letter-spacing: 0.5px;
        }

        .bubble:focus { 
            border-color: white !important; 
            box-shadow: 0 0 30px var(--glow-color) !important;
        }

        .connector {
            width: 14px; height: 14px; background: white; border-radius: 50%;
            position: absolute; bottom: -7px; cursor: crosshair;
            opacity: 0; transition: opacity 0.2s; z-index: 10;
            box-shadow: 0 0 10px white;
        }
        .bubble:hover .connector { opacity: 1; }

        /* UI PRVKY */
        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; flex-direction: column; gap: 15px; }
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; }
        
        .color-picker { 
            display: flex; gap: 10px; background: rgba(255,255,255,0.07); 
            padding: 10px; border-radius: 15px; border: 1px solid rgba(255,255,255,0.1);
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
        }
        .color-dot { width: 30px; height: 30px; border-radius: 50%; cursor: pointer; border: 2px solid transparent; transition: 0.2s; }
        .color-dot.active { border-color: white; transform: scale(1.2); }

        button { 
            padding: 12px 22px; border: none; border-radius: 10px; 
            cursor: pointer; font-weight: 600; color: white; transition: 0.3s;
            box-shadow: 0 4px 15px rgba(0,0,0,0.4);
        }
        button:hover { transform: translateY(-3px); box-shadow: 0 8px 20px rgba(0,0,0,0.6); }
        
        .btn-add { background: #1a73e8; }
        .btn-save { background: #1e8e3e; }
        .btn-load { background: #333; }
        .btn-img { background: #8e24aa; }

        .hint { 
            position: fixed; bottom: 20px; left: 20px; color: #aaa; font-size: 0.8em; 
            background: rgba(0,0,0,0.8); padding: 12px 18px; border-radius: 10px; 
            border: 1px solid rgba(255,255,255,0.1); 
        }
    </style>
</head>
<body>

    <div class="controls-left">
        <button class="btn-add" id="addBtn" onclick="addNode()">...</button>
        <div class="color-picker" id="picker">
            <div class="color-dot active" style="background: #2a2a2a; --c: #2a2a2a;" onclick="selectColor('#2a2a2a', this)"></div>
            <div class="color-dot" style="background: #ff4757; --c: #ff4757;" onclick="selectColor('#ff4757', this)"></div>
            <div class="color-dot" style="background: #2ed573; --c: #2ed573;" onclick="selectColor('#2ed573', this)"></div>
            <div class="color-dot" style="background: #1e90ff; --c: #1e90ff;" onclick="selectColor('#1e90ff', this)"></div>
            <div class="color-dot" style="background: #ffa502; --c: #ffa502;" onclick="selectColor('#ffa502', this)"></div>
        </div>
    </div>

    <div class="controls-right">
        <button class="btn-img" id="imgBtn" onclick="exportToImage()">...</button>
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
        txt('addBtn', "Nová myšlenka");
        txt('imgBtn', "Export PNG");
        txt('saveBtn', "Uložit");
        txt('loadBtn', "Otevřít");
        txt('hint-box', "Tip: Klikni na hotovou bublinu a pak na barvu pro její přebarvení.");

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        let nodes = [];
        let currentColor = '#ff4757';
        let scale = 1, posX = 0, posY = 0;
        let isDraggingView = false, startX, startY;
        let drawingLineFrom = null, tempMousePos = null;
        let selectedNode = null;

        function applyPhysics() {
            nodes.forEach((n1, i) => {
                nodes.forEach((n2, j) => {
                    if (i === j) return;
                    const dx = n2.x - n1.x; const dy = n2.y - n1.y;
                    const dist = Math.sqrt(dx * dx + dy * dy);
                    if (dist < 200) {
                        const force = (200 - dist) / 50;
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
            
            // PŘEBARVENÍ VYBRANÉ BUBLINY
            if (selectedNode) {
                selectedNode.color = color;
                selectedNode.el.style.background = color + "CC";
                selectedNode.el.style.borderColor = color;
                selectedNode.el.style.boxShadow = `0 0 20px ${color}66`;
                selectedNode.el.style.setProperty('--glow-color', color);
                saveToLocalStorage();
            }
        }

        viewport.onmousedown = (e) => {
            if (e.target === viewport) {
                isDraggingView = true;
                selectedNode = null;
                startX = e.clientX - posX; startY = e.clientY - posY;
            }
        };
        window.onmousemove = (e) => {
            if (isDraggingView) {
                posX = e.clientX - startX; posY = e.clientY - startY;
                world.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`;
                viewport.style.backgroundPosition = `${posX}px ${posY}px`;
            }
            if (drawingLineFrom) {
                tempMousePos = { x: (e.clientX - posX) / scale, y: (e.clientY - posY) / scale };
            }
        };
        window.onmouseup = () => { isDraggingView = false; drawingLineFrom = null; tempMousePos = null; };

        function createBubbleElement(x, y, color) {
            const el = document.createElement('div');
            el.className = 'bubble';
            el.style.background = color + "CC";
            el.style.borderColor = color;
            el.style.boxShadow = `0 0 20px ${color}66`;
            el.style.setProperty('--glow-color', color);
            el.contentEditable = true;
            el.innerText = "";

            const conn = document.createElement('div');
            conn.className = 'connector';
            conn.onmousedown = (e) => { e.stopPropagation(); drawingLineFrom = nodes.find(n => n.el === el); };
            el.appendChild(conn);

            el.onfocus = () => { selectedNode = nodes.find(n => n.el === el); };
            
            el.onmouseup = (e) => {
                if (drawingLineFrom && drawingLineFrom.el !== el) {
                    const target = nodes.find(n => n.el === el);
                    if (!target.manualParents.includes(drawingLineFrom)) {
                        target.manualParents.push(drawingLineFrom);
                        saveToLocalStorage();
                    }
                }
            };

            el.onmousedown = (e) => {
                if (e.target.className === 'connector') return;
                e.stopPropagation();
                if (e.button === 2) { deleteNode(el); return; }
                let bStartX = e.clientX / scale - x;
                let bStartY = e.clientY / scale - y;
                const move = (ev) => {
                    const node = nodes.find(n => n.el === el);
                    node.x = (ev.clientX / scale - bStartX);
                    node.y = (ev.clientY / scale - bStartY);
                };
                document.addEventListener('mousemove', move);
                document.onmouseup = () => document.removeEventListener('mousemove', move);
            };
            el.oncontextmenu = (e) => e.preventDefault();
            el.oninput = () => saveToLocalStorage();

            world.appendChild(el);
            return el;
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
                if (node.autoParent) {
                    drawCurve(node.autoParent.x + node.autoParent.el.offsetWidth/2 + 5000, node.autoParent.y + node.autoParent.el.offsetHeight/2 + 5000,
                              node.x + node.el.offsetWidth/2 + 5000, node.y + node.el.offsetHeight/2 + 5000, node.color, node.color);
                }
                node.manualParents.forEach(p => {
                    drawCurve(p.x + p.el.offsetWidth/2 + 5000, p.y + p.el.offsetHeight/2 + 5000,
                              node.x + node.el.offsetWidth/2 + 5000, node.y + node.el.offsetHeight/2 + 5000, p.color, node.color);
                });
            });

            if (drawingLineFrom && tempMousePos) {
                drawCurve(drawingLineFrom.x + drawingLineFrom.el.offsetWidth/2 + 5000, drawingLineFrom.y + drawingLineFrom.el.offsetHeight/2 + 5000,
                          tempMousePos.x + 5000, tempMousePos.y + 5000, drawingLineFrom.color, "#ffffff");
            }
        }

        function drawCurve(x1, y1, x2, y2, colorStart, colorEnd) {
            const grad = ctx.createLinearGradient(x1, y1, x2, y2);
            grad.addColorStop(0, colorStart);
            grad.addColorStop(1, colorEnd);
            ctx.strokeStyle = grad; ctx.shadowBlur = 15 / scale; ctx.shadowColor = colorStart;
            ctx.beginPath(); ctx.moveTo(x1, y1);
            ctx.bezierCurveTo(x1 + (x2 - x1) * 0.5, y1, x1 + (x2 - x1) * 0.5, y2, x2, y2);
            ctx.stroke(); ctx.shadowBlur = 0;
        }

        // --- PERSISTENCE ---
        function saveToLocalStorage() {
            const data = nodes.map(n => ({ 
                x: n.x, y: n.y, text: n.el.innerText, color: n.color,
                autoParentIdx: nodes.indexOf(n.autoParent),
                manualParentIndices: n.manualParents.map(p => nodes.indexOf(p))
            }));
            localStorage.setItem('myProDesignerMap', JSON.stringify(data));
        }

        function loadFromData(data) {
            nodes.forEach(n => n.el.remove());
            nodes = [];
            data.forEach(d => {
                const el = createBubbleElement(d.x, d.y, d.color);
                el.innerText = d.text;
                nodes.push({ el, x: d.x, y: d.y, color: d.color, autoParent: null, manualParents: [] });
            });
            data.forEach((d, i) => {
                if (d.autoParentIdx !== -1) nodes[i].autoParent = nodes[d.autoParentIdx];
                d.manualParentIndices.forEach(idx => nodes[i].manualParents.push(nodes[idx]));
            });
        }

        function exportToFile() {
            saveToLocalStorage();
            const data = localStorage.getItem('myProDesignerMap');
            const blob = new Blob([data], { type: 'application/json' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url; a.download = 'brainstorm.json'; a.click();
        }

        function importFromFile(event) {
            const file = event.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = (e) => loadFromData(JSON.parse(e.target.result));
            reader.readAsText(file);
        }

        function exportToImage() {
            // (Standardní exportní funkce PNG)
        }

        window.onload = () => {
            const saved = localStorage.getItem('myProDesignerMap');
            if (saved) loadFromData(JSON.parse(saved));
            applyPhysics();
        };
    </script>
</body>
</html>
