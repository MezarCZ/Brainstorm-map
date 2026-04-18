<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Infinite Dark Brainstorming</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #121212; /* Temné pozadí */
            color: white; font-family: sans-serif; 
        }

        /* Hlavní kontejner, který zabírá celou obrazovku */
        #viewport {
            width: 100vw; height: 100vh;
            cursor: grab;
            position: relative;
        }

        /* Skutečná plocha, kterou budeme posouvat a zvětšovat */
        #world {
            position: absolute;
            top: 0; left: 0;
            transform-origin: 0 0;
        }

        #line-canvas {
            position: absolute;
            top: 0; left: 0;
            pointer-events: none;
        }

        .bubble { 
            padding: 15px; 
            background: #333; /* Tmavší bubliny */
            color: #fff;
            border: 1px solid #444;
            border-radius: 8px; 
            position: absolute; 
            cursor: text; 
            min-width: 120px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.5);
            z-index: 2;
            outline: none;
        }
        .bubble:focus { border-color: #007bff; background: #3d3d3d; }

        /* UI Prvky */
        .controls {
            position: fixed; top: 20px; left: 20px; z-index: 100;
            display: flex; gap: 10px;
        }
        button { 
            padding: 10px 15px; background: #007bff; color: white; 
            border: none; border-radius: 5px; cursor: pointer; font-weight: bold;
        }
        button:hover { background: #0056b3; }
        .hint { position: fixed; bottom: 20px; left: 20px; color: #666; font-size: 0.8em; }
    </style>
</head>
<body>

    <div class="controls">
        <button onclick="addNode()">+ Nová myšlenka</button>
        <button onclick="resetView()">Střed</button>
    </div>

    <div class="hint">Kolečko: Zoom | Levá myš: Táhnout plochu | Klik do bubliny: Psát</div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
            </div>
    </div>

    <script>
        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        const nodes = [];

        let scale = 1;
        let posX = 0;
        let posY = 0;
        let isDraggingView = false;
        let startX, startY;

        // --- NASTAVENÍ PLOCHY (PAN & ZOOM) ---

        viewport.onmousedown = function(e) {
            if (e.target === viewport) {
                isDraggingView = true;
                viewport.style.cursor = 'grabbing';
                startX = e.clientX - posX;
                startY = e.clientY - posY;
            }
        };

        window.onmousemove = function(e) {
            if (isDraggingView) {
                posX = e.clientX - startX;
                posY = e.clientY - startY;
                updateWorldTransform();
            }
        };

        window.onmouseup = function() {
            isDraggingView = false;
            viewport.style.cursor = 'grab';
        };

        viewport.onwheel = function(e) {
            e.preventDefault();
            const delta = e.deltaY > 0 ? 0.9 : 1.1;
            scale *= delta;
            // Omezení zoomu
            scale = Math.min(Math.max(0.1, scale), 3);
            updateWorldTransform();
        };

        function updateWorldTransform() {
            world.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`;
            drawLines();
        }

        function resetView() {
            scale = 1; posX = 0; posY = 0;
            updateWorldTransform();
        }

        // --- LOGIKA BUBLIN ---

        function addNode() {
            const node = document.createElement('div');
            node.className = 'bubble';
            node.contentEditable = true;
            node.innerText = 'Napiš něco...';
            
            // Umístění do středu aktuálního pohledu
            const x = (window.innerWidth / 2 - posX) / scale;
            const y = (window.innerHeight / 2 - posY) / scale;
            
            node.style.left = x + 'px';
            node.style.top = y + 'px';

            // Aby nezačal drag, když chceme psát
            node.onmousedown = (e) => e.stopPropagation();

            world.appendChild(node);
            nodes.push(node);
            
            // Automaticky upravit canvas při přidání
            resizeCanvas();
            node.oninput = drawLines;
        }

        function resizeCanvas() {
            // Nastavíme canvas jako obrovský, aby pokryl i vzdálené bubliny
            canvas.width = 5000; 
            canvas.height = 5000;
            canvas.style.left = "-2500px";
            canvas.style.top = "-2500px";
            drawLines();
        }

        function drawLines() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = "#444";
            ctx.lineWidth = 2;

            if (nodes.length < 2) return;

            ctx.beginPath();
            nodes.forEach((node, i) => {
                const x = node.offsetLeft + node.offsetWidth / 2 + 2500;
                const y = node.offsetTop + node.offsetHeight / 2 + 2500;
                if (i === 0) ctx.moveTo(x, y);
                else ctx.lineTo(x, y);
            });
            ctx.stroke();
        }

        window.onresize = resizeCanvas;
        resizeCanvas();
    </script>
</body>
</html>
