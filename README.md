<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Moje Brainstorming Mapa</title>
    <style>
        body { font-family: sans-serif; text-align: center; background: #f0f2f5; margin: 0; padding: 20px; }
        
        /* Plocha pro mapu */
        #map-container {
            position: relative;
            width: 90%;
            height: 600px;
            margin: 20px auto;
            background: white;
            border: 2px solid #ddd;
            overflow: hidden; /* Aby bubliny neutíkaly ven */
        }

        /* Plátno pro čáry */
        #line-canvas {
            position: absolute;
            top: 0; left: 0;
            pointer-events: none; /* Aby čáry nepřekážely klikání na bubliny */
        }

        /* Styl bubliny */
        .bubble { 
            padding: 15px; 
            background: #ffad33; 
            border-radius: 10px; 
            position: absolute; 
            cursor: move; 
            min-width: 100px;
            box-shadow: 2px 4px 10px rgba(0,0,0,0.1);
            z-index: 2;
            outline: none;
        }

        .bubble:focus { border: 2px solid #333; }
        
        button { padding: 10px 20px; font-size: 16px; cursor: pointer; background: #007bff; color: white; border: none; border-radius: 5px; }
        button:hover { background: #0056b3; }
    </style>
</head>
<body>

    <h1>🧠 Brainstorming Mapa</h1>
    <button onclick="addNode()">Přidat myšlenku</button>

    <div id="map-container">
        <canvas id="line-canvas"></canvas>
    </div>

    <script>
        const container = document.getElementById('map-container');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        const nodes = [];

        // Přizpůsobení velikosti plátna
        function resizeCanvas() {
            canvas.width = container.clientWidth;
            canvas.height = container.clientHeight;
            drawLines();
        }
        window.onload = resizeCanvas;

        function addNode() {
            const node = document.createElement('div');
            node.className = 'bubble';
            node.contentEditable = true; // Tady povolujeme psaní!
            node.innerText = 'Klikni a piš...';
            
            // Počáteční pozice
            node.style.left = '50px';
            node.style.top = '50px';

            container.appendChild(node);
            nodes.push(node);

            // Logika tahání
            node.onmousedown = function(e) {
                // Pokud klikneš pro psaní, nechceme hned tahat
                if (document.activeElement === node) return;

                let shiftX = e.clientX - node.getBoundingClientRect().left;
                let shiftY = e.clientY - node.getBoundingClientRect().top;

                function moveAt(pageX, pageY) {
                    node.style.left = pageX - container.offsetLeft - shiftX + 'px';
                    node.style.top = pageY - container.offsetTop - shiftY + 'px';
                    drawLines(); // Překresli čáry při pohybu
                }

                function onMouseMove(e) { moveAt(e.pageX, e.pageY); }
                document.addEventListener('mousemove', onMouseMove);

                document.onmouseup = function() {
                    document.removeEventListener('mousemove', onMouseMove);
                    document.onmouseup = null;
                };
            };

            node.ondragstart = () => false;
            node.oninput = drawLines; // Překresli, když se změní délka textu
            drawLines();
        }

        // Funkce pro kreslení čar
        function drawLines() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.strokeStyle = "#aaa";
            ctx.lineWidth = 2;

            if (nodes.length < 2) return;

            ctx.beginPath();
            for (let i = 0; i < nodes.length; i++) {
                const rect = nodes[i].getBoundingClientRect();
                const containerRect = container.getBoundingClientRect();
                
                const x = rect.left - containerRect.left + rect.width / 2;
                const y = rect.top - containerRect.top + rect.height / 2;

                if (i === 0) ctx.moveTo(x, y);
                else ctx.lineTo(x, y);
            }
            ctx.stroke();
        }
    </script>
</body>
</html>
