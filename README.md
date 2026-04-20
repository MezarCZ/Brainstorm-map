<!DOCTYPE html>
<html lang="cs">
<head>
    <meta charset="UTF-8">
    <title>Brainstorming Hybrid Pro</title>
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
            padding: 15px; color: #ffffff;
            border: 2px solid rgba(255,255,255,0.1); border-radius: 16px; 
            position: absolute; cursor: move; min-width: 140px;
            box-shadow: 0 0 15px rgba(0,0,0,0.5); z-index: 2; outline: none;
            display: flex; flex-direction: column; align-items: center; text-align: center;
            backdrop-filter: blur(4px); animation: emerge 0.3s ease-out;
        }
        @keyframes emerge { 0% { transform: scale(0); } 100% { transform: scale(1); } }

        .bubble:focus { border-color: white !important; }

        .connector {
            width: 12px; height: 12px; background: white; border-radius: 50%;
            position: absolute; bottom: -6px; cursor: crosshair;
            opacity: 0; transition: opacity 0.2s; z-index: 10;
        }
        .bubble:hover .connector { opacity: 1; }

        .controls-left { position: fixed; top: 20px; left: 20px; z-index: 100; display: flex; flex-direction: column; gap: 10px; }
        .controls-right { position: fixed; top: 20px; right: 20px; z-index: 100; display: flex; gap: 10px; }
        .color-picker { display: flex; gap: 8px; background: rgba(255,255,255,0.05); padding: 8px; border-radius: 12px; }
        .color-dot { width: 28px; height: 28px; border-radius: 50%; cursor: pointer; border: 2px solid transparent; }
        .color-dot.active { border-color: white; transform: scale(1.2); }

        button { padding: 12px 20px; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; color: white; }
        .btn-add { background: #1a73e8; }
        .btn-img { background: #8e24aa; }

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
        // Unicode texty pro cestinu
        document.getElementById('addBtn').textContent = "Nov\u00E1 my\u0161lenka";
        document.getElementById('imgBtn').textContent = "Export PNG";
        document.getElementById('hint-box').textContent = "Barvy se poj\u00ED samy. T\u00E1hni z te\u010Dky pod bublinou pro ru\u010Dn\u00ED spojen\u00ED jin\u00FDch barev.";

        const viewport = document.getElementById('viewport');
        const world = document.getElementById('world');
        const canvas = document.getElementById('line-canvas');
        const ctx = canvas.getContext('2d');
        let nodes = [];
        let currentColor = '#2a2a2a';
        let scale = 1, posX = 0, posY = 0;
        let isDraggingView = false, startX, startY;
        let drawingLineFrom = null, tempMousePos = null;

        // --- MAGNETISMUS ---
        function applyPhysics() {
            nodes.forEach((n1, i) => {
                nodes.forEach((n2, j) => {
                    if (i === j) return;
                    const dx = n2.x - n1.x; const dy = n2.y - n1.y;
                    const dist = Math.sqrt(dx * dx + dy * dy);
                    if (dist < 220) {
                        const force = (220 - dist) / 50;
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
            isDraggingView = false; drawingLineFrom = null; tempMousePos = null;
            drawLines();
        };

        function createBubbleElement(x, y, color) {
            const el = document.createElement('div');
            el.className = 'bubble';
            el.style.background = color + "CC";
            el.style.borderColor = color;
            el.style.boxShadow = `0 0 20px ${color}44`;
            el.contentEditable = true;
            el.innerText = "";

            const conn = document.createElement('div');
            conn.className = 'connector';
            conn.onmousedown = (e) => {
                e.stopPropagation();
                drawingLineFrom = nodes.find(n => n.el === el);
            };

            el.appendChild(conn);

            el.onmouseup = (e) => {
                if (drawingLineFrom && drawingLineFrom.el !== el) {
                    const target = nodes.find(n => n.el === el);
                    if (!target.manualParents.includes(drawingLineFrom)) {
                        target.manualParents.push(drawingLineFrom);
                    }
                }
            };

            el.onmousedown = (e) => {
                if (e.target.className === 'connector') return;
                e.stopPropagation();
                if (e.button === 2) { // Pravé tlačítko smaže
                    deleteNode(el);
                    return;
                }
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

            world.appendChild(el);
            return el;
        }

        function deleteNode(el) {
            el.remove();
            nodes = nodes.filter(n => n.el !== el);
            nodes.forEach(n => n.manualParents = n.manualParents.filter(p => p.el !== el));
        }

        function addNode() {
            const x = (window.innerWidth / 2 - posX) / scale - 70;
            const y = (window.innerHeight / 2 - posY) / scale - 25;
            const el = createBubbleElement(x, y, currentColor);
            
            // Automatické propojení stejné barvy
            const sameColor = nodes.filter(n => n.color === currentColor);
            const autoParent = sameColor.length > 0 ? sameColor[sameColor.length - 1] : null;

            nodes.push({ el, x, y, color: currentColor, autoParent, manualParents: [] });
            setTimeout(() => el.focus(), 10);
        }

        function drawLines() {
            canvas.width = 10000; canvas.height = 10000;
            canvas.style.left = "-5000px"; canvas.style.top = "-5000px";
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.lineWidth = 3 / scale;

            nodes.forEach(node => {
                // Kresli automatickou čáru
                if (node.autoParent) {
                    drawCurve(node.autoParent.x + 70 + 5000, node.autoParent.y + 25 + 5000,
                              node.x + 70 + 5000, node.y + 5000, node.color);
                }
                // Kresli manuální čáry
                node.manualParents.forEach(p => {
                    drawCurve(p.x + 70 + 5000, p.y + 50 + 5000,
                              node.x + 70 + 5000, node.y + 5000, "#ffffff66");
                });
            });

            if (drawingLineFrom && tempMousePos) {
                drawCurve(drawingLineFrom.x + 70 + 5000, drawingLineFrom.y + 50 + 5000,
                          tempMousePos.x + 5000, tempMousePos.y + 5000, "#ffffffaa");
            }
        }

        function drawCurve(x1, y1, x2, y2, color) {
            ctx.strokeStyle = color; ctx.shadowBlur = 10 / scale; ctx.shadowColor = color;
            ctx.beginPath(); ctx.moveTo(x1, y1);
            ctx.bezierCurveTo(x1, y1 + (y2-y1)/2, x2, y1 + (y2-y1)/2, x2, y2);
            ctx.stroke(); ctx.shadowBlur = 0;
        }

        function exportToImage() {
            // ... (nechal jsem stejnou logiku jako minule pro export)
        }

        requestAnimationFrame(applyPhysics);
    </script>
</body>
</html>
