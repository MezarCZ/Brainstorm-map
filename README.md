<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8"> <title>Infinite Dark Brainstorming CZ</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #121212; 
            color: white; 
            /* Použijeme fonty, které umí česky na 100 % */
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
        }

        #viewport { width: 100vw; height: 100vh; cursor: grab; position: relative; }
        #world { position: absolute; top: 0; left: 0; transform-origin: 0 0; }
        #line-canvas { position: absolute; top: 0; left: 0; pointer-events: none; }

        .bubble { 
            padding: 15px; 
            background: #2a2a2a; 
            color: #ffffff;
            border: 2px solid #444;
            border-radius: 12px; 
            position: absolute; 
            cursor: move; /* Kurzor naznačuje, že s tím jde hýbat */
            min-width: 140px;
            box-shadow: 0 6px 20px rgba(0,0,0,0.6);
            z-index: 2;
            outline: none;
            transition: border-color 0.2s;
        }
        
        /* Styl při psaní */
        .bubble:focus { border-color: #007bff; background: #333; cursor: text; }

        .controls { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; gap: 10px; }
        button { 
            padding: 12px 18px; background: #007bff; color: white; 
            border: none; border-radius: 6px; cursor: pointer; font-weight: bold;
            box-shadow: 0 2px 10px rgba(0,0,0,0.3);
        }
        button:hover { background: #0056b3; }
        .hint { position: fixed; bottom: 20px; left: 20px; color: #888; font-size: 0.9em; background: rgba(0,0,0,0.5); padding: 5px 10px; border-radius: 4px; }
    </style>
</head>
<body>

    <div class="controls">
        <button onclick="addNode()">+ Nový nápad (háčky/čárky)</button>
        <button onclick="resetView()">Střed</button>
    </div>

    <div class="hint">Levá myš na pozadí: Posun plochy | Levá myš na bublinu: Přesun bubliny | Klik a psaní: Č, Š, Ž... OK!</div>

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

        // --- PAN & ZOOM ---
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
            scale = Math.min(Math.max(0.2, scale), 3);
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

        // --- BUBLINY ---
        function addNode() {
            const node = document.createElement('div');
            node.className = 'bubble';
            node.contentEditable = true;
            node.innerText = 'Příliš žluťoučký kůň...'; // Český testovací text
            
            const x = (window.innerWidth / 2 - posX) / scale;
            const y = (window.innerHeight / 2 - posY) / scale;
            
            node.style.left = x + 'px';
            node.style.top = y + 'px';

            // PŘETAHOVÁNÍ BUBLIN V RÁMCI SVĚTA
            node.onmousedown = function(e) {
                e.stopPropagation(); // Zabránit posunu celé plochy
                
                // Pokud už v bublině píšeme, nechceme ji hned stěhovat
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
            drawLines();
        }

        function drawLines() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = "#444";
            ctx.lineWidth = 2 / scale; // Aby čára nebyla tlustá při zoomu

            canvas.width = 10000; // Ještě větší rezerva
            canvas.height = 10000;
            canvas.style.left = "-5000px";
            canvas.style.top = "-5000px";

            if (nodes.length < 2) return;

            ctx.beginPath();
            nodes.forEach((node, i) => {
                const x = node.offsetLeft + node.offsetWidth / 2 + 5000;
                const y = node.offsetTop + node.offsetHeight / 2 + 5000;
                if (i === 0) ctx.moveTo(x, y);
                else ctx.lineTo(x, y);
            });
            ctx.stroke();
        }

        window.onload = () => { resetView(); drawLines(); };
    </script>
</body>
</html>
