<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Infinite Brainstorming</title>
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

        .controls { position: fixed; top: 20px; left: 20px; z-index: 100; }
        button { 
            padding: 12px 18px; background: #007bff; color: white; 
            border: none; border-radius: 6px; cursor: pointer; font-weight: bold;
        }
        button:hover { background: #0056b3; }
        .hint { position: fixed; bottom: 20px; left: 20px; color: #888; font-size: 0.85em; background: rgba(0,0,0,0.5); padding: 8px 12px; border-radius: 4px; }
    </style>
</head>
<body>

    <div class="controls">
        <button id="addBtn" onclick="addNode()">Nová myšlenka</button>
    </div>

    <div id="hint-box" class="hint"></div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
        </div>
    </div>

    <script>
        // Oprava diakritiky přes JavaScript, aby nebyly otazníky
        document.getElementById('addBtn').innerText = "Nov\u00E1 my\u0161lenka";
        document.getElementById('hint-box').innerText = "Lev\u00E1 my\u0161 na pozad\u00ED: Posun plochy | Lev\u00E1 my\u0161 na bublinu: P\u0159esun | Prav\u00E9 tla\u010D\u00EDtko na bublinu: Smazat";

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        const nodes = [];

        let scale = 1, posX = 0, posY = 0;
        let isDraggingView = false, startX, startY;

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

        window.onmouseup = () => { isDraggingView = false; viewport.style.cursor = 'grab'; };

        viewport.onwheel = function(e) {
            e.preventDefault();
            const delta = e.deltaY > 0 ? 0.9 : 1.1;
            scale = Math.min(Math.max(0.2, scale * delta), 3);
            updateWorldTransform();
        };

        function updateWorldTransform() {
            world.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`;
            drawLines();
        }

        function addNode() {
            const node = document.createElement('div');
            node.className = 'bubble';
            node.contentEditable = true;
            node.innerText = '';
            
            const x = (window.innerWidth / 2 - posX) / scale;
            const y = (window.innerHeight / 2 - posY) / scale;
            node.style.left = x + 'px';
            node.style.top = y + 'px';

            // MAZÁNÍ (Pravé tlačítko)
            node.oncontextmenu = function(e) {
                e.preventDefault();
                node.remove();
                nodes.splice(nodes.indexOf(node), 1);
                drawLines();
            };

            // POHYB BUBLINY
            node.onmousedown = function(e) {
                if (e.button !== 0) return; // Jen levé tlačítko
                e.stopPropagation();
                if (document.activeElement === node) return;

                let bStartX = e.clientX / scale - parseInt(node.style.left);
                let bStartY = e.clientY / scale - parseInt(node.style.top);

                function onMoveBubble(ev) {
                    node.style.left = (ev.clientX / scale - bStartX) + 'px';
                    node.style.top = (ev.clientY / scale - bStartY) + 'px';
                    drawLines();
                }

                document.addEventListener('mousemove', onMoveBubble);
                document.onmouseup = function() {
                    document.removeEventListener('mousemove', onMoveBubble);
                    document.onmouseup = null;
                };
            };

            world.appendChild(node);
            nodes.push(node);
            node.oninput = drawLines;
            setTimeout(() => node.focus(), 10);
            drawLines();
        }

        function drawLines() {
            canvas.width = 10000; canvas.height = 10000;
            canvas.style.left = "-5000px"; canvas.style.top = "-5000px";
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = "#444"; ctx.lineWidth = 2 / scale;

            if (nodes.length < 2) return;
            ctx.beginPath();
            nodes.forEach((node, i) => {
                const x = node.offsetLeft + node.offsetWidth / 2 + 5000;
                const y = node.offsetTop + node.offsetHeight / 2 + 5000;
                if (i === 0) ctx.moveTo(x, y); else ctx.lineTo(x, y);
            });
            ctx.stroke();
        }

        window.onload = () => { updateWorldTransform(); drawLines(); };
    </script>
</body>
</html>
