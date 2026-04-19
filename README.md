<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Brainstorming Neon Pro</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #0a0a0b; color: white; 
            font-family: 'Segoe UI', Roboto, sans-serif; 
        }
        #viewport { width: 100vw; height: 100vh; cursor: grab; position: relative; }
        #world { position: absolute; top: 0; left: 0; transform-origin: 0 0; }
        #line-canvas { position: absolute; top: 0; left: 0; pointer-events: none; }

        .bubble { 
            padding: 15px; color: #ffffff;
            border: 2px solid rgba(255,255,255,0.1); border-radius: 16px; 
            position: absolute; cursor: move; min-width: 140px; min-height: 25px;
            box-shadow: 0 0 15px rgba(0,0,0,0.5); z-index: 2; outline: none;
            display: flex; align-items: center; justify-content: center; text-align: center;
            transition: transform 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275), box-shadow 0.3s;
            animation: emerge 0.4s ease-out;
            backdrop-filter: blur(4px);
        }
        
        @keyframes emerge {
            0% { transform: scale(0); opacity: 0; }
            100% { transform: scale(1); opacity: 1; }
        }

        .bubble:focus { border-color: white; box-shadow: 0 0 20px rgba(255,255,255,0.4); }

        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; flex-direction: column; gap: 10px; }
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; flex-wrap: wrap; justify-content: flex-end; }

        .color-picker { display: flex; gap: 8px; background: rgba(255,255,255,0.05); padding: 8px; border-radius: 12px; border: 1px solid rgba(255,255,255,0.1); }
        .color-dot { width: 28px; height: 28px; border-radius: 50%; cursor: pointer; border: 2px solid transparent; transition: 0.2s; position: relative; }
        .color-dot.active { border-color: white; transform: scale(1.2); }

        button { 
            padding: 12px 20px; background: #1a73e8; color: white; 
            border: none; border-radius: 8px; cursor: pointer; font-weight: bold;
            transition: 0.2s; box-shadow: 0 4px 10px rgba(0,0,0,0.3);
        }
        button:hover { background: #1557b0; transform: translateY(-2px); }
        .btn-save { background: #1e8e3e; }
        .btn-img { background: #8e24aa; }

        .hint { position: fixed; bottom: 20px; left: 20px; color: #666; font-size: 0.85em; background: rgba(0,0,0,0.7); padding: 10px 15px; border-radius: 8px; border: 1px solid rgba(255,255,255,0.05); }
    </style>
</head>
<body>

    <div class="controls-left">
        <button id="addBtn" onclick="addNode()">...</button>
        <div class="color-picker" id="picker">
            <div class="color-dot active" style="background: #2a2a2a; box-shadow: 0 0 10px #2a2a2a;" onclick="selectColor('#2a2a2a', this)"></div>
            <div class="color-dot" style="background: #ff4757; box-shadow: 0 0 10px #ff4757;" onclick="selectColor('#ff4757', this)"></div>
            <div class="color-dot" style="background: #2ed573; box-shadow: 0 0 10px #2ed573;" onclick="selectColor('#2ed573', this)"></div>
            <div class="color-dot" style="background: #1e90ff; box-shadow: 0 0 10px #1e90ff;" onclick="selectColor('#1e90ff', this)"></div>
            <div class="color-dot" style="background: #ffa502; box-shadow: 0 0 10px #ffa502;" onclick="selectColor('#ffa502', this)"></div>
            <div class="color-dot" style="background: #5f27cd; box-shadow: 0 0 10px #5f27cd;" onclick="selectColor('#5f27cd', this)"></div>
        </div>
    </div>

    <div class="controls-right">
        <button class="btn-img" id="imgBtn" onclick="exportToImage()">...</button>
        <button class="btn-save" id="saveBtn" onclick="exportToFile()">...</button>
        <button id="loadBtn" onclick="document.getElementById('fileInput').click()">...</button>
        <input type="file" id="fileInput" style="display:none" onchange="importFromFile(event)">
    </div>

    <div id="hint-box" class="hint">...</div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
        </div>
    </div>

    <script>
        // Unicode texty
        document.getElementById('addBtn').textContent = "Nov\u00E1 my\u0161lenka";
        document.getElementById('imgBtn').textContent = "Ulo\u017Eit jako obr\u00E1zek";
        document.getElementById('saveBtn').textContent = "Ulo\u017Eit projekt";
        document.getElementById('loadBtn').textContent = "Otev\u0159\u00EDt projekt";
        document.getElementById('hint-box').textContent = "Prav\u00E9 tla\u010D\u00EDtko ma\u017Ee. Dr\u017E a t\u00E1hni pro pohyb v map\u011B.";

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        let nodes = [];
        let currentColor = '#2a2a2a';

        let scale = 1, posX = 0, posY = 0;
        let isDraggingView = false, startX, startY;

        function selectColor(color, el) {
            currentColor = color;
            document.querySelectorAll('.color-dot').forEach(d => d.classList.remove('active'));
            el.classList.add('active');
        }

        viewport.onmousedown = (e) => {
            if (e.target === viewport) {
                isDraggingView = true;
                startX = e.clientX - posX; startY = e.clientY - posY;
            }
        };
        window.onmousemove = (e) => {
            if (isDraggingView) {
                posX = e.clientX - startX; posY = e.clientY - startY;
                updateWorldTransform();
            }
        };
        window.onmouseup = () => isDraggingView = false;
        viewport.onwheel = (e) => {
            e.preventDefault();
            const delta = e.deltaY > 0 ? 0.9 : 1.1;
            scale = Math.min(Math.max(0.2, scale * delta), 3);
            updateWorldTransform();
        };

        function updateWorldTransform() {
            world.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`;
            drawLines();
        }

        function createBubbleElement(x, y, text, color) {
            const node = document.createElement('div');
            node.className = 'bubble';
            node.contentEditable = true;
            node.innerText = text || "";
            node.style.left = x + 'px';
            node.style.top = y + 'px';
            node.style.background = color + "CC"; // Průhlednost
            node.style.boxShadow = `0 0 20px ${color}66`;
            node.style.borderColor = color;

            node.oncontextmenu = (e) => {
                e.preventDefault();
                node.remove();
                nodes = nodes.filter(n => n.el !== node);
                saveToLocalStorage();
                drawLines();
            };

            node.onmousedown = (e) => {
                if (e.button !== 0) return;
                e.stopPropagation();
                if (document.activeElement === node) return;
                let bStartX = e.clientX / scale - parseInt(node.style.left);
                let bStartY = e.clientY / scale - parseInt(node.style.top);
                const move = (ev) => {
                    node.style.left = (ev.clientX / scale - bStartX) + 'px';
                    node.style.top = (ev.clientY / scale - bStartY) + 'px';
                    drawLines();
                };
                document.addEventListener('mousemove', move);
                document.onmouseup = () => {
                    document.removeEventListener('mousemove', move);
                    saveToLocalStorage();
                };
            };

            node.oninput = () => { drawLines(); saveToLocalStorage(); };
            world.appendChild(node);
            return node;
        }

        function addNode() {
            const x = (window.innerWidth / 2 - posX) / scale - 70;
            const y = (window.innerHeight / 2 - posY) / scale - 25;
            const color = currentColor;
            const el = createBubbleElement(x, y, '', color);
            
            const sameColorNodes = nodes.filter(n => n.color === color);
            const parent = sameColorNodes.length > 0 ? sameColorNodes[sameColorNodes.length - 1] : null;

            nodes.push({ el, color, parent });
            setTimeout(() => el.focus(), 10);
            drawLines();
            saveToLocalStorage();
        }

        function drawLines() {
            canvas.width = 10000; canvas.height = 10000;
            canvas.style.left = "-5000px"; canvas.style.top = "-5000px";
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.lineWidth = 3 / scale;

            nodes.forEach(node => {
                if (node.parent) {
                    const p = node.parent.el;
                    const c = node.el;
                    
                    const x1 = p.offsetLeft + p.offsetWidth / 2 + 5000;
                    const y1 = p.offsetTop + p.offsetHeight / 2 + 5000;
                    const x2 = c.offsetLeft + c.offsetWidth / 2 + 5000;
                    const y2 = c.offsetTop + c.offsetHeight / 2 + 5000;

                    // GLOW EFEKT ČÁRY
                    ctx.shadowBlur = 10 / scale;
                    ctx.shadowColor = node.color;
                    ctx.strokeStyle = node.color;
                    
                    // BEZIEROVA KŘIVKA (S-curve)
                    ctx.beginPath();
                    ctx.moveTo(x1, y1);
                    // Kontrolní body pro zaoblení
                    const cp1x = x1 + (x2 - x1) * 0.5;
                    const cp1y = y1;
                    const cp2x = x1 + (x2 - x1) * 0.5;
                    const cp2y = y2;
                    ctx.bezierCurveTo(cp1x, cp1y, cp2x, cp2y, x2, y2);
                    ctx.stroke();
                    ctx.shadowBlur = 0;
                }
            });
        }

        function exportToImage() {
            if (nodes.length === 0) return;
            let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
            nodes.forEach(n => {
                const x = parseInt(n.el.style.left);
                const y = parseInt(n.el.style.top);
                minX = Math.min(minX, x); minY = Math.min(minY, y);
                maxX = Math.max(maxX, x + n.el.offsetWidth);
                maxY = Math.max(maxY, y + n.el.offsetHeight);
            });

            const padding = 100;
            const exportCanvas = document.createElement('canvas');
            const eCtx = exportCanvas.getContext('2d');
            exportCanvas.width = (maxX - minX) + padding * 2;
            exportCanvas.height = (maxY - minY) + padding * 2;

            eCtx.fillStyle = "#0a0a0b";
            eCtx.fillRect(0, 0, exportCanvas.width, exportCanvas.height);

            eCtx.lineWidth = 3;
            nodes.forEach(node => {
                if (node.parent) {
                    const p = node.parent.el;
                    const c = node.el;
                    const x1 = parseInt(p.style.left) - minX + p.offsetWidth / 2 + padding;
                    const y1 = parseInt(p.style.top) - minY + p.offsetHeight / 2 + padding;
                    const x2 = parseInt(c.style.left) - minX + c.offsetWidth / 2 + padding;
                    const y2 = parseInt(c.style.top) - minY + c.offsetHeight / 2 + padding;
                    
                    eCtx.shadowBlur = 10; eCtx.shadowColor = node.color;
                    eCtx.strokeStyle = node.color;
                    eCtx.beginPath();
                    eCtx.moveTo(x1, y1);
                    const cp1x = x1 + (x2 - x1) * 0.5;
                    const cp2x = x1 + (x2 - x1) * 0.5;
                    eCtx.bezierCurveTo(cp1x, y1, cp2x, y2, x2, y2);
                    eCtx.stroke();
                }
            });

            nodes.forEach(node => {
                const x = parseInt(node.el.style.left) - minX + padding;
                const y = parseInt(node.el.style.top) - minY + padding;
                const w = node.el.offsetWidth; const h = node.el.offsetHeight;
                eCtx.fillStyle = node.color;
                eCtx.shadowBlur = 15; eCtx.shadowColor = node.color;
                eCtx.beginPath(); eCtx.roundRect(x, y, w, h, 16); eCtx.fill();
                eCtx.shadowBlur = 0;
                eCtx.fillStyle = "white"; eCtx.font = "bold 16px sans-serif";
                eCtx.textAlign = "center"; eCtx.textBaseline = "middle";
                eCtx.fillText(node.el.innerText, x + w / 2, y + h / 2);
            });

            const link = document.createElement('a');
            link.download = 'neon-mapa.png';
            link.href = exportCanvas.toDataURL();
            link.click();
        }

        function saveToLocalStorage() {
            const data = nodes.map(n => ({ 
                x: n.el.style.left, y: n.el.style.top, text: n.el.innerText, color: n.color,
                parentIdx: nodes.indexOf(n.parent) 
            }));
            localStorage.setItem('myNeonMap', JSON.stringify(data));
        }

        function exportToFile() {
            saveToLocalStorage();
            const data = localStorage.getItem('myNeonMap');
            const blob = new Blob([data], { type: 'application/json' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url; a.download = 'projekt-neon.json'; a.click();
        }

        function importFromFile(event) {
            const file = event.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = (e) => loadFromData(JSON.parse(e.target.result));
            reader.readAsText(file);
        }

        function loadFromData(data) {
            nodes.forEach(n => n.el.remove());
            nodes = [];
            data.forEach(d => {
                const el = createBubbleElement(parseInt(d.x), parseInt(d.y), d.text, d.color);
                nodes.push({ el, color: d.color, parent: null });
            });
            data.forEach((d, i) => { if (d.parentIdx !== -1 && nodes[d.parentIdx]) nodes[i].parent = nodes[d.parentIdx]; });
            drawLines();
            saveToLocalStorage();
        }

        window.onload = () => {
            const saved = localStorage.getItem('myNeonMap');
            if (saved) loadFromData(JSON.parse(saved));
            updateWorldTransform();
        };
    </script>
</body>
</html>
