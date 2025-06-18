# Jpg-to-pdf
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Image to PDF Converter</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
  <style>
    :root {
      --bg-color: #f0f2f5;
      --text-color: #000;
      --card-color: #fff;
    }
    [data-theme="dark"] {
      --bg-color: #121212;
      --text-color: #f1f1f1;
      --card-color: #1e1e1e;
    }
    body {
      font-family: Arial, sans-serif;
      background: var(--bg-color);
      color: var(--text-color);
      margin: 0;
      padding: 20px;
      transition: all 0.3s ease;
    }
    .container {
      max-width: 800px;
      margin: auto;
      background: var(--card-color);
      padding: 30px;
      border-radius: 12px;
      box-shadow: 0 5px 15px rgba(0,0,0,0.1);
    }
    h2 {
      text-align: center;
    }
    .drop-zone {
      border: 2px dashed #0077b6;
      padding: 40px;
      text-align: center;
      cursor: pointer;
      border-radius: 10px;
      background: #e3f2fd;
      margin-top: 20px;
    }
    .drop-zone.dragover {
      background: #d0eaff;
    }
    .preview {
      display: flex;
      flex-wrap: wrap;
      gap: 15px;
      margin-top: 20px;
    }
    .preview img {
      width: 100px;
      height: 100px;
      object-fit: cover;
      border-radius: 8px;
      border: 1px solid #ccc;
      cursor: move;
    }
    label, select, button {
      display: block;
      margin-top: 20px;
      font-size: 16px;
    }
    button {
      padding: 10px 20px;
      background: #0077b6;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
    }
    button:hover {
      background: #005f87;
    }
    .toggle-dark {
      float: right;
      margin-bottom: 10px;
    }
    .sort-options {
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
  </style>
</head>
<body data-theme="light">

<div class="container">
  <button class="toggle-dark" onclick="toggleTheme()">Toggle Dark Mode</button>
  <h2>Image (JPG, PNG, WEBP) to PDF Converter</h2>

  <div class="drop-zone" id="dropZone">Drag & drop images or click to select</div>
  <input type="file" id="imageInput" accept="image/jpeg,image/png,image/webp" multiple style="display:none">

  <div class="sort-options">
    <label for="sortOrder">Sort images by:</label>
    <select id="sortOrder" onchange="sortImages()">
      <option value="none">Manual Order</option>
      <option value="name">Filename</option>
      <option value="date">Upload Time</option>
    </select>
  </div>

  <div class="preview" id="previewArea"></div>

  <label for="orientation">Select Page Orientation:</label>
  <select id="orientation">
    <option value="portrait">Portrait (A4)</option>
    <option value="landscape">Landscape (A4)</option>
  </select>

  <button onclick="generatePDF()">Download PDF</button>
</div>

<script>
  const dropZone = document.getElementById('dropZone');
  const imageInput = document.getElementById('imageInput');
  const previewArea = document.getElementById('previewArea');
  let imageFiles = [];

  dropZone.addEventListener('click', () => imageInput.click());
  dropZone.addEventListener('dragover', (e) => {
    e.preventDefault();
    dropZone.classList.add('dragover');
  });
  dropZone.addEventListener('dragleave', () => dropZone.classList.remove('dragover'));
  dropZone.addEventListener('drop', (e) => {
    e.preventDefault();
    dropZone.classList.remove('dragover');
    handleFiles(e.dataTransfer.files);
  });
  imageInput.addEventListener('change', () => handleFiles(imageInput.files));

  function handleFiles(files) {
    [...files].forEach(file => {
      if (['image/jpeg', 'image/png', 'image/webp'].includes(file.type)) {
        imageFiles.push({ file, timestamp: Date.now() });
      }
    });
    renderPreviews();
  }

  function renderPreviews() {
    previewArea.innerHTML = "";
    imageFiles.forEach((obj, index) => {
      const reader = new FileReader();
      reader.onload = e => {
        const img = document.createElement('img');
        img.src = e.target.result;
        img.setAttribute('data-index', index);
        previewArea.appendChild(img);
        makeSortable();
      };
      reader.readAsDataURL(obj.file);
    });
  }

  function makeSortable() {
    let dragged;
    previewArea.querySelectorAll('img').forEach(img => {
      img.draggable = true;
      img.addEventListener('dragstart', (e) => dragged = e.target);
      img.addEventListener('dragover', (e) => e.preventDefault());
      img.addEventListener('drop', (e) => {
        e.preventDefault();
        if (dragged !== e.target) {
          let i = parseInt(dragged.getAttribute('data-index'));
          let j = parseInt(e.target.getAttribute('data-index'));
          [imageFiles[i], imageFiles[j]] = [imageFiles[j], imageFiles[i]];
          renderPreviews();
        }
      });
    });
  }

  function sortImages() {
    const type = document.getElementById('sortOrder').value;
    if (type === 'name') {
      imageFiles.sort((a, b) => a.file.name.localeCompare(b.file.name));
    } else if (type === 'date') {
      imageFiles.sort((a, b) => a.timestamp - b.timestamp);
    }
    renderPreviews();
  }

  async function generatePDF() {
    if (imageFiles.length === 0) {
      alert("Please upload at least one image.");
      return;
    }

    const { jsPDF } = window.jspdf;
    const orientation = document.getElementById('orientation').value;
    const pdf = new jsPDF({ orientation: orientation, unit: 'mm', format: 'a4' });
    const pageWidth = orientation === 'portrait' ? 210 : 297;
    const pageHeight = orientation === 'portrait' ? 297 : 210;

    for (let i = 0; i < imageFiles.length; i++) {
      const imgData = await readFileAsDataURL(imageFiles[i].file);
      const img = new Image();
      img.src = imgData;
      await new Promise(resolve => {
        img.onload = function () {
          const canvas = document.createElement('canvas');
          const ctx = canvas.getContext('2d');
          canvas.width = img.width * 0.5;
          canvas.height = img.height * 0.5;
          ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
          const compressedData = canvas.toDataURL('image/jpeg', 0.7);

          let width = pageWidth;
          let height = (img.height * width) / img.width;
          if (height > pageHeight) {
            height = pageHeight;
            width = (img.width * height) / img.height;
          }
          if (i > 0) pdf.addPage();
          pdf.addImage(compressedData, 'JPEG', (pageWidth - width) / 2, (pageHeight - height) / 2, width, height);
          resolve();
        };
      });
    }
    pdf.save("converted-images.pdf");
  }

  function readFileAsDataURL(file) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = e => resolve(e.target.result);
      reader.onerror = e => reject(e);
      reader.readAsDataURL(file);
    });
  }

  function toggleTheme() {
    const body = document.body;
    body.setAttribute('data-theme', body.getAttribute('data-theme') === 'dark' ? 'light' : 'dark');
  }
</script>

</body>
</html>
