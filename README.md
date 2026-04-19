<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Infinite Brainstorming Color Edition</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #121212; color: white; 
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
        }
        #viewport { width: 100vw; height: 100vh; cursor: grab; position: relative; }
        #world { position: absolute; top: 0; left: 0; transform-origin: 0 0; }
        #line-canvas { position: absolute; top: 0; left: 0; pointer-events: none; }

        .bubble { 
            padding: 15px; color: #ffffff;
            border: 2px solid rgba(255,255,255,0.2); border-radius: 12px; 
            position: absolute; cursor: move; min-width: 140px; min-height: 20px;
            box-shadow: 0 6px 20px rgba(0,0,0,0.6); z-index: 2; outline: none;
        }
        .bubble:focus { border-color: white; box-shadow: 0 0 15px rgba(255,255,255,0.3); cursor: text; }

        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; flex-direction: column; gap: 10px; }
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; }

        .color-picker { display: flex; gap: 5px; background: rgba(255,255,255,0.1); padding: 5px; border-radius: 8px; }
        .color-dot { width: 25px; height: 25px; border-radius: 50%; cursor: pointer; border: 2px solid transparent; }
        .color-dot.active { border-color: white; transform: scale(1.1); }

        button { 
            padding: 12px 18px; background: #007bff; color: white; 
            border: none; border-radius: 6px; cursor: pointer; font-weight: bold;
        }
        button:hover { background: #0056b3; }
        .btn-save { background: #28a745; }

        .hint { position: fixed; bottom: 20px; left: 20px; color: #888; font-size: 0.85em; background: rgba(0,0,0,0.5); padding: 8px 12px; border-radius: 4px; }
    </style>
</head>
<body>

    <div class="controls-left">
        <button id="addBtn" onclick="addNode()">Nová myšlenka</button>
        <div class="color-picker" id="picker">
            <div class="color-dot active" style="background: #2a2a2a;" onclick="selectColor('#2a2a2a', this)"></div>
            <div class="color-dot" style="background: #e74c3c;" onclick="selectColor('#e74c3c', this)"></div>
            <div class="color-dot" style="background: #3498db;" onclick="selectColor('#3498db', this)"></div>
            <div class="color-dot" style="background: #2ecc71;" onclick="selectColor('#2ecc71', this)"></div>
            <div class="color-dot" style="background: #f1c40f;" onclick="selectColor('#f1c40f', this)"></div>
            <div class="color-dot" style="background: #9b59b6;" onclick="selectColor('#9b59b6', this)"></div>
        </div>
    </div>

    <div class="controls-right">
        <button class="btn-save" id="saveBtn" onclick="exportToFile()">Uložit projekt</button>
        <button id="loadBtn" onclick="document.getElementById('fileInput').click()">Otevřít projekt</button>
        <input type="file" id="fileInput" style="display:none" onchange="importFromFile(event)">
    </div>

    <div id="hint-box" class="hint"></div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
        </div>
    </div>

    <script>
        document.getElementById('addBtn').innerText = "Nov\u00E1 my\u0161lenka";
        document.getElementById('saveBtn').innerText = "Ulo\u017Eit projekt";
        document.getElementById('loadBtn').innerText = "Otev\u0159\u00EDt projekt";
        document.getElementById('hint-box').innerText = "Lev\u00E1 my\u0161 na pozad\u00ED: Posun plochy | Prav\u00E9 tla\u010D\u00EDtko: Smazat | Vyber barvu a p\u0159idej my\u0161lenku.";

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

        function createBubbleElement(x, y, text = '', color) {
            const node = document.createElement('div');
            node.className = 'bubble';
            node.contentEditable = true;
            node.innerText = text;
            node.style.left = x + 'px';
            node.style.top = y + 'px';
            node.style.background = color || currentColor;
            node.dataset.color = color || currentColor;

            node.oncontextmenu = (e) => {
                e.preventDefault();
                node.remove();
                nodes = nodes.filter(n => n !== node);
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
            const x = (window.innerWidth / 2 - posX) / scale;
            const y = (window.innerHeight / 2 - posY) / scale;
            const node = createBubbleElement(x, y);
            nodes.push(node);
            setTimeout(() => node.focus(), 10);
            drawLines();
            saveToLocalStorage();
        }

        function drawLines() {
            canvas.width = 10000; canvas.height = 10000;
            canvas.style.left = "-5000px"; canvas.style.top = "-5000px";
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.lineWidth = 3 / scale;

            if (nodes.length < 2) return;

            for (let i = 1; i < nodes.length; i++) {
                const prev = nodes[i-1];
                const curr = nodes[i];
                
                const x1 = prev.offsetLeft + prev.offsetWidth / 2 + 5000;
                const y1 = prev.offsetTop + prev.offsetHeight / 2 + 5000;
                const x2 = curr.offsetLeft + curr.offsetWidth / 2 + 5000;
                const y2 = curr.offsetTop + curr.offsetHeight / 2 + 5000;

                // Čára má barvu bubliny, ke které vede
                ctx.strokeStyle = curr.dataset.color;
                ctx.beginPath();
                ctx.moveTo(x1, y1);
                ctx.lineTo(x2, y2);
                ctx.stroke();
            }
        }

        function saveToLocalStorage() {
            const data = nodes.map(n => ({ x: n.style.left, y: n.style.top, text: n.innerText, color: n.dataset.color }));
            localStorage.setItem('myColorMap', JSON.stringify(data));
        }

        function exportToFile() {
            const data = nodes.map(n => ({ x: n.style.left, y: n.style.top, text: n.innerText, color: n.dataset.color }));
            const blob = new Blob([JSON.stringify(data)], { type: 'application/json' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url; a.download = 'barevna-mapa.json'; a.click();
        }

        function importFromFile(event) {
            const file = event.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = (e) => loadFromData(JSON.parse(e.target.result));
            reader.readAsText(file);
        }

        function loadFromData(data) {
            nodes.forEach(n => n.remove());
            nodes = [];
            data.forEach(d => nodes.push(createBubbleElement(parseInt(d.x), parseInt(d.y), d.text, d.color)));
            drawLines();
            saveToLocalStorage();
        }

        window.onload = () => {
            const saved = localStorage.getItem('myColorMap');
            if (saved) loadFromData(JSON.parse(saved));
            updateWorldTransform();
        };
    </script>
</body>
</html>
