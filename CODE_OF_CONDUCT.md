<!DOCTYPE html>
<html lang="tr">
<head>
  <meta charset="UTF-8">
  <title>Kova Aracılı Boyama Uygulaması</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body {
      font-family: sans-serif;
      margin: 0;
      padding: 10px;
    }

    #controls {
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      margin-bottom: 10px;
      align-items: center;
    }

    canvas {
      width: 100%;
      max-width: 100%;
      border: 2px solid #333;
      touch-action: none;
    }

    button, select, input[type="range"], input[type="color"] {
      font-size: 14px;
      padding: 5px;
    }
  </style>
</head>
<body>
  <h2>Kova Aracılı Boyama Uygulaması</h2>

  <input type="file" id="imageLoader" accept="image/*">

  <div id="controls">
    Renk: <input type="color" id="colorPicker" value="#ff0000">
    Kalınlık: <input type="range" id="brushSize" min="1" max="30" value="5">
    Uç: 
    <select id="brushShape">
      <option value="round">Yuvarlak</option>
      <option value="square">Kare</option>
    </select>
    <button id="eraser">Silgi</button>
    <button id="fill">Kova</button>
    <button id="undo">Geri Al</button>
    <button id="clear">Temizle</button>
    <button id="save">Kaydet</button>
  </div>

  <canvas id="paintCanvas" width="800" height="600"></canvas>

  <script>
    const canvas = document.getElementById('paintCanvas');
    const ctx = canvas.getContext('2d');
    const imageLoader = document.getElementById('imageLoader');
    const colorPicker = document.getElementById('colorPicker');
    const brushSize = document.getElementById('brushSize');
    const brushShape = document.getElementById('brushShape');
    const eraserBtn = document.getElementById('eraser');
    const fillBtn = document.getElementById('fill');
    const undoBtn = document.getElementById('undo');
    const saveBtn = document.getElementById('save');
    const clearBtn = document.getElementById('clear');

    let painting = false;
    let erasing = false;
    let filling = false;
    const history = [];

    function saveState() {
      history.push(canvas.toDataURL());
      if (history.length > 20) history.shift();
    }

    function getPos(e) {
      const rect = canvas.getBoundingClientRect();
      if (e.touches) {
        return {
          x: e.touches[0].clientX - rect.left,
          y: e.touches[0].clientY - rect.top
        };
      }
      return { x: e.offsetX, y: e.offsetY };
    }

    function draw(e) {
      if (!painting || filling) return;
      const pos = getPos(e);
      ctx.lineWidth = brushSize.value;
      ctx.lineCap = brushShape.value;
      ctx.strokeStyle = erasing ? '#ffffff' : colorPicker.value;

      ctx.lineTo(pos.x, pos.y);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(pos.x, pos.y);
    }

    imageLoader.addEventListener('change', function (e) {
      const reader = new FileReader();
      reader.onload = function (event) {
        const img = new Image();
        img.onload = function () {
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
          saveState();
        };
        img.src = event.target.result;
      };
      reader.readAsDataURL(e.target.files[0]);
    });

    canvas.addEventListener('mousedown', (e) => {
      if (filling) {
        const pos = getPos(e);
        saveState();
        floodFill(pos.x, pos.y, hexToRgba(colorPicker.value));
      } else {
        painting = true;
        saveState();
        draw(e);
      }
    });

    canvas.addEventListener('mouseup', () => {
      painting = false;
      ctx.beginPath();
    });

    canvas.addEventListener('mousemove', draw);
    canvas.addEventListener('touchstart', (e) => {
      if (filling) {
        const pos = getPos(e);
        saveState();
        floodFill(pos.x, pos.y, hexToRgba(colorPicker.value));
      } else {
        painting = true;
        saveState();
        draw(e);
      }
    });
    canvas.addEventListener('touchend', () => {
      painting = false;
      ctx.beginPath();
    });

    canvas.addEventListener('touchmove', draw);

    eraserBtn.addEventListener('click', () => {
      erasing = !erasing;
      filling = false;
      eraserBtn.textContent = erasing ? 'Fırçaya Dön' : 'Silgi';
      fillBtn.textContent = 'Kova';
    });

    fillBtn.addEventListener('click', () => {
      filling = !filling;
      erasing = false;
      fillBtn.textContent = filling ? 'Fırçaya Dön' : 'Kova';
      eraserBtn.textContent = 'Silgi';
    });

    undoBtn.addEventListener('click', () => {
      if (history.length > 0) {
        const last = history.pop();
        const img = new Image();
        img.onload = () => ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        img.src = last;
      }
    });

    saveBtn.addEventListener('click', () => {
      const link = document.createElement('a');
      link.download = 'boyama.png';
      link.href = canvas.toDataURL();
      link.click();
    });

    clearBtn.addEventListener('click', () => {
      saveState();
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    });

    // --- KOVA ARACI (FLOOD FILL) ---

    function floodFill(startX, startY, fillColor) {
      const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      const data = imgData.data;
      const targetColor = getColorAt(startX, startY, data);
      if (!colorsMatch(targetColor, fillColor)) {
        const pixelStack = [[startX, startY]];
        while (pixelStack.length) {
          const [x, y] = pixelStack.pop();
          const idx = (y * canvas.width + x) * 4;
          const currentColor = getColorAt(x, y, data);
          if (colorsMatch(currentColor, targetColor)) {
            setColorAt(idx, fillColor, data);
            if (x > 0) pixelStack.push([x - 1, y]);
            if (x < canvas.width - 1) pixelStack.push([x + 1, y]);
            if (y > 0) pixelStack.push([x, y - 1]);
            if (y < canvas.height - 1) pixelStack.push([x, y + 1]);
          }
        }
        ctx.putImageData(imgData, 0, 0);
      }
    }

    function getColorAt(x, y, data) {
      const index = (y * canvas.width + x) * 4;
      return [
        data[index],
        data[index + 1],
        data[index + 2],
        data[index + 3]
      ];
    }

    function setColorAt(index, color, data) {
      data[index] = color[0];
      data[index + 1] = color[1];
      data[index + 2] = color[2];
      data[index + 3] = 255;
    }

    function colorsMatch(a, b) {
      return Math.abs(a[0] - b[0]) < 30 &&
             Math.abs(a[1] - b[1]) < 30 &&
             Math.abs(a[2] - b[2]) < 30;
    }

    function hexToRgba(hex) {
      const bigint = parseInt(hex.slice(1), 16);
      return [(bigint >> 16) & 255, (bigint >> 8) & 255, bigint & 255, 255];
    }
  </script>
</body>
</html>
