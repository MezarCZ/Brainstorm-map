<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Brainstorming Ultimate Pro</title>
    <style>
        body { 
            margin: 0; padding: 0; overflow: hidden; 
            background-color: #0a0a0b; color: white; 
            font-family: 'Segoe UI', Roboto, sans-serif; 
            user-select: none;
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
            transition: transform 0.2s, box-shadow 0.3s;
            animation: emerge 0.4s ease-out;
            backdrop-filter: blur(4px);
        }
        
        @keyframes emerge { 0% { transform: scale(0); } 100% { transform: scale(1); } }

        .bubble:focus { border-color: white !important; box-shadow: 0 0 20px rgba(255,255,255,0.4); }

        /* Poznámka pod čarou */
        .note-area {
            width: 100%; margin-top: 8px; padding-top: 8px;
            border-top: 1px solid rgba(255,255,255,0.2);
            font-size: 0.75em; color: rgba(255,255,255,0.6);
            font-style: italic; display: none;
        }
        .bubble:focus .note-area, .bubble:not(:empty) .note-area { display: block; }

        /* Konektor pro manuální spojování */
        .connector {
            width: 12px; height: 12px; background: white; border-radius: 50%;
            position: absolute; bottom: -6px; cursor: crosshair;
            opacity: 0; transition: opacity 0.2s;
        }
        .bubble:hover .connector { opacity: 1; }

        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; flex-direction: column; gap: 10px; }
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; }

        .color-picker { display: flex; gap: 8px; background: rgba(255,255,255,0.05); padding: 8px; border-radius: 12px; }
        .color-dot { width: 28px; height: 28px; border-radius: 50%; cursor: pointer; border: 2px solid transparent; }
        .color-dot.active { border-color: white; transform: scale(1.2); }

        button { padding: 12px 20px; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; }
        .btn-add { background: #1a73e8; color: white; }
        .btn-img { background: #8e24aa; color: white; }

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
        <button class="btn-img" id="imgBtn" onclick="exportToImage()">...</button>
    </div>

    <div id="hint-box" class="hint">...</div>

    <div id="viewport">
        <div id="world">
            <canvas id="line-canvas"></canvas>
        </div>
    </div>

    <script>
        const cz = (t) => decodeURIComponent(escape(t));
        document.getElementById('addBtn').textContent = "Nov\u00E1 my\u0161lenka";
        document.getElementById('imgBtn').textContent = "Export PNG";
        document.getElementById('hint-box').textContent = "T\u00E1hni z te\u010Dky pod bublinou pro ru\u010Dn\u00ED spojen\u00ED. Magnety udr\u017Euj\u00ED po\u0159\u00E1dek.";

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        let nodes = [];
        let currentColor = '#ff4757';

        let scale = 1, posX = 0, posY = 0;
        let isDraggingView = false, startX, startY;
        let drawingLineFrom = null, tempMousePos = null;

        // --- MAGNETISMUS (FYZIKA) ---
        function applyPhysics() {
            for (let i = 0; i < nodes.length; i++) {
                for (let j = i + 1; j < nodes.length; j++) {
                    const n1 = nodes[i]; const n2 = nodes[j];
                    const dx = n2.x - n1.x; const dy = n2.y - n1.y;
                    const dist = Math.sqrt(dx * dx + dy * dy);
                    const minDist = 200;

                    if (dist < minDist) {
                        const force = (minDist - dist) / 40;
                        const fx = (dx / dist) * force;
                        const fy = (dy / dist) * force;
                        n1.x -= fx; n1.y -= fy;
                        n2.x += fx; n2.y += fy;
                    }
                }
                nodes[i].el.style.left = nodes[i].x + 'px';
                nodes[i].el.style.top = nodes[i].y + 'px';
            }
            drawLines();
            requestAnimationFrame(applyPhysics);
        }

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
                world.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`;
            }
            if (drawingLineFrom) {
                tempMousePos = { x: (e.clientX - posX) / scale, y: (e.clientY - posY) / scale };
                drawLines();
            }
        };
        window.onmouseup = () => {
            isDraggingView = false;
            drawingLineFrom = null; tempMousePos = null;
            drawLines();
        };

        function createBubbleElement(x, y, color) {
            const nodeEl = document.createElement('div');
            nodeEl.className = 'bubble';
            nodeEl.style.background = color + "CC";
            nodeEl.style.borderColor = color;
            nodeEl.style.boxShadow = `0 0 20px ${color}44`;

            const title = document.createElement('div');
            title.contentEditable = true;
            title.innerText = "N\u00E1pad...";
            title.style.outline = "none";

            const note = document.createElement('div');
            note.className = 'note-area';
            note.contentEditable = true;
            note.innerText = "Pozn\u00E1mka...";

            const conn = document.createElement('div');
            conn.className = 'connector';
            conn.onmousedown = (e) => {
                e.stopPropagation();
                drawingLineFrom = nodes.find(n => n.el === nodeEl);
            };

            nodeEl.appendChild(title);
            nodeEl.appendChild(note);
            nodeEl.appendChild(conn);

            nodeEl.onmouseup = (e) => {
                if (drawingLineFrom && drawingLineFrom.el !== nodeEl) {
                    const targetNode = nodes.find(n => n.el === nodeEl);
                    if (!targetNode.parents.includes(drawingLineFrom)) {
                        targetNode.parents.push(drawingLineFrom);
                    }
                }
            };

            nodeEl.onmousedown = (e) => {
                if (e.target.className === 'connector') return;
                e.stopPropagation();
                let bStartX = e.clientX / scale - x;
                let bStartY = e.clientY / scale - y;
                const move = (ev) => {
                    const node = nodes.find(n => n.el === nodeEl);
                    node.x = (ev.clientX / scale - bStartX);
                    node.y = (ev.clientY / scale - bStartY);
                };
                document.addEventListener('mousemove', move);
                document.onmouseup = () => document.removeEventListener('mousemove', move);
            };

            world.appendChild(nodeEl);
            return nodeEl;
        }

        function addNode() {
            const x = (window.innerWidth / 2 - posX) / scale - 70;
            const y = (window.innerHeight / 2 - posY) / scale - 25;
            const el = createBubbleElement(x, y, currentColor);
            nodes.push({ el, x, y, color: currentColor, parents: [] });
        }

        function drawLines() {
            canvas.width = 10000; canvas.height = 10000;
            canvas.style.left = "-5000px"; canvas.style.top = "-5000px";
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.lineWidth = 3 / scale;

            nodes.forEach(node => {
                node.parents.forEach(parent => {
                    drawCurve(parent.x + parent.el.offsetWidth/2 + 5000, parent.y + parent.el.offsetHeight + 5000,
                              node.x + node.el.offsetWidth/2 + 5000, node.y + 5000, node.color);
                });
            });

            if (drawingLineFrom && tempMousePos) {
                drawCurve(drawingLineFrom.x + drawingLineFrom.el.offsetWidth/2 + 5000, drawingLineFrom.y + drawingLineFrom.el.offsetHeight + 5000,
                          tempMousePos.x + 5000, tempMousePos.y + 5000, drawingLineFrom.color);
            }
        }

        function drawCurve(x1, y1, x2, y2, color) {
            ctx.strokeStyle = color; ctx.shadowBlur = 10; ctx.shadowColor = color;
            ctx.beginPath(); ctx.moveTo(x1, y1);
            ctx.bezierCurveTo(x1, y1 + (y2-y1)/2, x2, y1 + (y2-y1)/2, x2, y2);
            ctx.stroke(); ctx.shadowBlur = 0;
        }

        requestAnimationFrame(applyPhysics);
    </script>
</body>
</html>
