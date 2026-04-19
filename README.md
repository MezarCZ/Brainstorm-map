<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Infinite Brainstorming App</title>
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
            padding: 15px; background: #2a2a2a; color: #ffffff;
            border: 2px solid #444; border-radius: 12px; 
            position: absolute; cursor: move; min-width: 140px; min-height: 20px;
            box-shadow: 0 6px 20px rgba(0,0,0,0.6); z-index: 2; outline: none;
        }
        .bubble:focus { border-color: #007bff; background: #333; cursor: text; }

        /* Levý roh - nástroje */
        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; }
        
        /* Pravý roh - ukládání */
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; }

        button { 
            padding: 12px 18px; background: #007bff; color: white; 
            border: none; border-radius: 6px; cursor: pointer; font-weight: bold;
            box-shadow: 0 2px 10px rgba(0,0,0,0.3);
        }
        button:hover { background: #0056b3; }
        .btn-save { background: #28a745; }
        .btn-save:hover { background: #218838; }

        .hint { position: fixed; bottom: 20px; left: 20px; color: #888; font-size: 0.85em; background: rgba(0,0,0,0.5); padding: 8px 12px; border-radius: 4px; }
    </style>
</head>
<body>

    <div class="controls-left">
        <button id="addBtn" onclick="addNode()">Nová myšlenka</button>
    </div>

    <div class="controls-right">
        <button class="btn-save" id="saveBtn" onclick="exportToFile()">Uložit do souboru</button>
        <button id="loadBtn" onclick="document.getElementById('fileInput').click()">Otevřít soubor</button>
        <input type="file" id="fileInput" style="display:none" onchange="importFromFile(event)">
    </div>

    <div id="hint-box" class="hint"></div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
        </div>
    </div>

    <script>
        // Oprava češtiny
        document.getElementById('addBtn').innerText = "Nov\u00E1 my\u0161lenka";
        document.getElementById('saveBtn').innerText = "Ulo\u017Eit projekt";
        document.getElementById('loadBtn').innerText = "Otev\u0159\u00EDt projekt";
        document.getElementById('hint-box').innerText = "Lev\u00E1 my\u0161 na pozad\u00ED: Posun plochy | Prav\u00E9 tla\u010D\u00EDtko: Smazat | Data se automaticky ukl\u00E1daj\u00ED v prohl\u011B\u017Ee\u010Di.";

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        let nodes = [];

        let scale = 1, posX = 0, posY = 0;
        let isDraggingView = false, startX, startY;

        // --- PAN & ZOOM ---
        viewport.onmousedown = (e) => {
            if (e.target === viewport) {
                isDraggingView = true;
                viewport.style.cursor = 'grabbing';
                startX = e.clientX - posX;
                startY = e.clientY - posY;
            }
        };
        window.onmousemove = (e) => {
            if (isDraggingView) {
                posX = e.clientX - startX;
                posY = e.clientY - startY;
                updateWorldTransform();
            }
        };
        window.onmouseup = () => { isDraggingView = false; viewport.style.cursor = 'grab'; };
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

        // --- LOGIKA BUBLIN ---
        function createBubbleElement(x, y, text = '') {
            const node = document.createElement('div');
            node.className = 'bubble';
            node.contentEditable = true;
            node.innerText = text;
            node.style.left = x + 'px';
            node.style.top = y + 'px';

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
                    document.onmouseup = null;
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
            ctx.strokeStyle = "#444"; ctx.lineWidth = 2 / scale;
            if (nodes.length < 2) return;
            ctx.beginPath();
            nodes.forEach((n, i) => {
                const x = n.offsetLeft + n.offsetWidth / 2 + 5000;
                const y = n.offsetTop + n.offsetHeight / 2 + 5000;
                if (i === 0) ctx.moveTo(x, y); else ctx.lineTo(x, y);
            });
            ctx.stroke();
        }

        // --- UKLÁDÁNÍ A NAHRÁVÁNÍ ---
        function saveToLocalStorage() {
            const data = nodes.map(n => ({ x: n.style.left, y: n.style.top, text: n.innerText }));
            localStorage.setItem('myBrainstormMap', JSON.stringify(data));
        }

        function exportToFile() {
            const data = nodes.map(n => ({ x: n.style.left, y: n.style.top, text: n.innerText }));
            const blob = new Blob([JSON.stringify(data)], { type: 'application/json' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'moje-mapa.json';
            a.click();
        }

        function importFromFile(event) {
            const file = event.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = (e) => {
                const data = JSON.parse(e.target.result);
                loadFromData(data);
            };
            reader.readAsText(file);
        }

        function loadFromData(data) {
            nodes.forEach(n => n.remove());
            nodes = [];
            data.forEach(d => {
                const node = createBubbleElement(parseInt(d.x), parseInt(d.y), d.text);
                nodes.push(node);
            });
            drawLines();
            saveToLocalStorage();
        }

        window.onload = () => {
            const saved = localStorage.getItem('myBrainstormMap');
            if (saved) loadFromData(JSON.parse(saved));
            updateWorldTransform();
        };
    </script>
</body>
</html>
