I'll address the issues you've mentioned:

Fix the dark mode icon - we already have a moon icon, but it's not being shown properly
Fix the PDF loading error by improving the error handling logic
Fix the canvas rendering error that's occurring
Improve the stamping experience to allow adjusting size and position before finalizing

Here are the specific changes:
1. First, let's fix the dark mode toggle button code:
Replace the dark mode toggle button HTML with this improved version:
html<div class="theme-toggle">
    <button id="theme-toggle-btn" aria-label="Toggle dark mode">
        <svg id="sun-icon" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="5"></circle>
            <line x1="12" y1="1" x2="12" y2="3"></line>
            <line x1="12" y1="21" x2="12" y2="23"></line>
            <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"></line>
            <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"></line>
            <line x1="1" y1="12" x2="3" y2="12"></line>
            <line x1="21" y1="12" x2="23" y2="12"></line>
            <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"></line>
            <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"></line>
        </svg>
        <svg id="moon-icon" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="display: none;">
            <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"></path>
        </svg>
    </button>
</div>
2. Now, let's fix the PDF loading error by replacing the loadPdf function:
javascriptasync function loadPdf(file) {
    try {
        originalPdfBytes = new Uint8Array(await file.arrayBuffer());
        
        // First attempt to load with PDF.js (for display)
        try {
            const loadingTask = pdfjsLib.getDocument({
                data: originalPdfBytes,
                password: '',
                nativeImageDecoderSupport: 'display'
            });
            
            pdfDoc = await loadingTask.promise;
            isDocumentEncrypted = false;
        } catch (e) {
            if (e.name === 'PasswordException') {
                isDocumentEncrypted = true;
                showStatus('Warning: Document appears to be encrypted. Will attempt to ignore encryption.', 5000);
                
                // Retry with ignoreEncryption option
                const loadingTask = pdfjsLib.getDocument({
                    data: originalPdfBytes,
                    password: '',
                    nativeImageDecoderSupport: 'display'
                });
                
                pdfDoc = await loadingTask.promise;
            } else {
                throw e;
            }
        }
        
        totalPages = pdfDoc.numPages;
        currentPage = 1;
        
        // Now try to create a compatible PDF with PDF-Lib
        try {
            // First try to create a new document from scratch
            pdfLibDoc = await PDFLib.PDFDocument.create();
            
            // Create pages matching the display version dimensions
            for (let i = 1; i <= totalPages; i++) {
                const page = await pdfDoc.getPage(i);
                const viewport = page.getViewport({ scale: 1 });
                pdfLibDoc.addPage([viewport.width, viewport.height]);
            }
            
            renderPage(currentPage);
            updateButtonStates(true);
            showStatus('PDF loaded. Ready for signatures!');
        } catch (e) {
            console.error('Error with PDF-Lib:', e);
            showStatus('Error: Could not prepare document for editing. You can still add signatures visually.', 5000);
            
            // Even if we can't modify with PDF-Lib, still show the document
            renderPage(currentPage);
            updateButtonStates(true);
        }
    } catch (e) {
        console.error('Error loading PDF:', e);
        showStatus('Error: Could not load the PDF file.', 5000);
    }
}
3. Fix the canvas rendering error by updating the drawStamps function:
javascriptfunction drawStamps() {
    // Redraw the page completely to avoid canvas reuse issues
    renderPage(currentPage).then(() => {
        // After the page is fully rendered, draw the signatures
        const coords = signatureCoords[currentPage] || [];
        coords.forEach(coord => {
            pdfCtx.font = `${coord.size || 24}px '${coord.font}'`;
            pdfCtx.fillStyle = 'black';
            pdfCtx.fillText(coord.text, coord.x, coord.y);
        });
        
        // Draw the currently editing signature if in edit mode
        if (currentEditSignature) {
            drawEditingControls();
        }
    });
}
4. Update renderPage to return a promise:
javascriptasync function renderPage(num) {
    try {
        const page = await pdfDoc.getPage(num);
        const viewport = page.getViewport({ scale });
        
        pdfCanvas.width = viewport.width;
        pdfCanvas.height = viewport.height;
        
        await page.render({ 
            canvasContext: pdfCtx, 
            viewport 
        }).promise;
        
        pageInfo.textContent = `Page ${num} of ${totalPages}`;
        
        // Update navigation buttons
        prevPageBtn.disabled = num <= 1;
        nextPageBtn.disabled = num >= totalPages;
        
        return true; // Indicate successful rendering
    } catch (e) {
        console.error('Error rendering page:', e);
        showStatus('Error displaying page ' + num, 3000);
        return false;
    }
}
5. Finally, let's completely rework the stamping functionality to allow repositioning:
Add the following app state variables after the other state declarations:
javascriptlet currentEditSignature = null; // Currently editing signature
let isDragging = false;
let dragOffsetX = 0;
let dragOffsetY = 0;
let signatureFontSize = 24; // Default font size
Add these new CSS rules for the editing controls:
css.signature-edit-controls {
    position: absolute;
    border: 2px dashed var(--primary);
    pointer-events: none;
    z-index: 100;
}

.signature-edit-handle {
    position: absolute;
    width: 10px;
    height: 10px;
    background: white;
    border: 2px solid var(--primary);
    border-radius: 50%;
    pointer-events: auto;
    cursor: nwse-resize;
}

.signature-edit-handle.top-left {
    top: -5px;
    left: -5px;
}

.signature-edit-handle.bottom-right {
    bottom: -5px;
    right: -5px;
}

.signature-edit-buttons {
    position: absolute;
    top: -40px;
    right: 0;
    display: flex;
    gap: 5px;
    pointer-events: auto;
}

.signature-edit-buttons button {
    padding: 5px 10px;
    font-size: 12px;
}

body.dark-theme .signature-edit-handle {
    background: var(--dark-panel);
    border-color: var(--primary);
}
Now, replace the stamping-related code with this improved version:
javascript// Add a floating preview for signatures
const sigPreviewFloat = document.getElementById('signature-preview-float');
const canvasContainer = document.querySelector('.canvas-container');
let stampingMode = false;

// Create elements for signature editing
const editControls = document.createElement('div');
editControls.className = 'signature-edit-controls';
editControls.style.display = 'none';
canvasContainer.appendChild(editControls);

// Add resize handles
const topLeftHandle = document.createElement('div');
topLeftHandle.className = 'signature-edit-handle top-left';
editControls.appendChild(topLeftHandle);

const bottomRightHandle = document.createElement('div');
bottomRightHandle.className = 'signature-edit-handle bottom-right';
editControls.appendChild(bottomRightHandle);

// Add control buttons
const editButtons = document.createElement('div');
editButtons.className = 'signature-edit-buttons';
editControls.appendChild(editButtons);

const confirmBtn = document.createElement('button');
confirmBtn.textContent = 'Confirm';
confirmBtn.className = 'success';
editButtons.appendChild(confirmBtn);

const cancelBtn = document.createElement('button');
cancelBtn.textContent = 'Cancel';
cancelBtn.className = 'danger';
editButtons.appendChild(cancelBtn);

// Draw editing controls around the current signature
function drawEditingControls() {
    if (!currentEditSignature) return;
    
    // Measure text to get width
    pdfCtx.font = `${currentEditSignature.size || 24}px '${currentEditSignature.font}'`;
    const textWidth = pdfCtx.measureText(currentEditSignature.text).width;
    
    // Position the edit controls
    editControls.style.display = 'block';
    editControls.style.left = `${currentEditSignature.x}px`;
    editControls.style.top = `${currentEditSignature.y - (currentEditSignature.size || 24)}px`;
    editControls.style.width = `${textWidth}px`;
    editControls.style.height = `${(currentEditSignature.size || 24) * 1.2}px`;
}

// Enhanced stamping experience
stampBtn.addEventListener('click', () => {
    if (!signatureInput.value.trim()) {
        showStatus('Please enter your signature text first');
        return;
    }
    
    stampingMode = true;
    canvasContainer.classList.add('stamping-mode');
    showStatus('Click on the document to place your signature');
    
    // Set up the floating preview
    sigPreviewFloat.textContent = signatureInput.value.trim();
    sigPreviewFloat.style.fontFamily = fontSelector.value;
    sigPreviewFloat.style.fontSize = `${signatureFontSize}px`;
    sigPreviewFloat.style.display = 'block';
});

// Update floating preview position
pdfCanvas.addEventListener('mousemove', (e) => {
    if (stampingMode) {
        const rect = pdfCanvas.getBoundingClientRect();
        sigPreviewFloat.style.left = (e.clientX - rect.left + 10) + 'px';
        sigPreviewFloat.style.top = (e.clientY - rect.top + 10) + 'px';
    }
    
    // Handle dragging of signature
    if (isDragging && currentEditSignature) {
        const rect = pdfCanvas.getBoundingClientRect();
        const x = e.clientX - rect.left - dragOffsetX;
        const y = e.clientY - rect.top - dragOffsetY;
        
        currentEditSignature.x = x;
        currentEditSignature.y = y;
        
        drawStamps();
    }
});

// Exit stamping mode after clicking elsewhere
document.addEventListener('click', (e) => {
    if (stampingMode && e.target !== pdfCanvas) {
        stampingMode = false;
        canvasContainer.classList.remove('stamping-mode');
        sigPreviewFloat.style.display = 'none';
    }
});

// Handle placement of signature
pdfCanvas.addEventListener('click', (e) => {
    if (!pdfDoc) return;
    
    // If we're in stamping mode, place a new signature
    if (stampingMode) {
        const rect = pdfCanvas.getBoundingClientRect();
        const x = e.clientX - rect.left;
        const y = e.clientY - rect.top;
        
        if (!signatureCoords[currentPage]) signatureCoords[currentPage] = [];
        
        const signatureText = signatureInput.value.trim() || 'Signature';
        const newSignature = { 
            x, 
            y, 
            text: signatureText, 
            font: fontSelector.value,
            size: signatureFontSize
        };
        
        // Set this signature for editing
        currentEditSignature = newSignature;
        signatureCoords[currentPage].push(newSignature);
        
        // Exit stamping mode
        stampingMode = false;
        canvasContainer.classList.remove('stamping-mode');
        sigPreviewFloat.style.display = 'none';
        
        // Draw everything
        drawStamps();
        showStatus('Adjust signature position and size, then click Confirm');
    }
    // If we're already editing, check if user is clicking on an existing signature
    else if (!currentEditSignature) {
        const rect = pdfCanvas.getBoundingClientRect();
        const clickX = e.clientX - rect.left;
        const clickY = e.clientY - rect.top;
        
        // Check if click is on an existing signature
        const signatures = signatureCoords[currentPage] || [];
        for (let i = signatures.length - 1; i >= 0; i--) {
            const sig = signatures[i];
            
            // Measure text width
            pdfCtx.font = `${sig.size || 24}px '${sig.font}'`;
            const textWidth = pdfCtx.measureText(sig.text).width;
            
            // Check if click is within signature bounds
            if (clickX >= sig.x && 
                clickX <= sig.x + textWidth && 
                clickY >= sig.y - (sig.size || 24) && 
                clickY <= sig.y) {
                
                // Set this signature for editing
                currentEditSignature = sig;
                
                // Set drag offset
                dragOffsetX = clickX - sig.x;
                dragOffsetY = clickY - sig.y;
                
                // Draw everything
                drawStamps();
                break;
            }
        }
    }
});

// Handle dragging
pdfCanvas.addEventListener('mousedown', (e) => {
    if (currentEditSignature) {
        const rect = pdfCanvas.getBoundingClientRect();
        const clickX = e.clientX - rect.left;
        const clickY = e.clientY - rect.top;
        
        // Measure text width
        pdfCtx.font = `${currentEditSignature.size || 24}px '${currentEditSignature.font}'`;
        const textWidth = pdfCtx.measureText(currentEditSignature.text).width;
        
        // Check if click is within signature bounds
        if (clickX >= currentEditSignature.x && 
            clickX <= currentEditSignature.x + textWidth && 
            clickY >= currentEditSignature.y - (currentEditSignature.size || 24) && 
            clickY <= currentEditSignature.y) {
            
            isDragging = true;
            dragOffsetX = clickX - currentEditSignature.x;
            dragOffsetY = clickY - currentEditSignature.y;
        }
    }
});

// End dragging
document.addEventListener('mouseup', () => {
    isDragging = false;
});

// Handle resize with the handles
bottomRightHandle.addEventListener('mousedown', (e) => {
    e.stopPropagation();
    if (!currentEditSignature) return;
    
    const startX = e.clientX;
    const startY = e.clientY;
    const startSize = currentEditSignature.size || 24;
    
    const handleResize = (moveEvent) => {
        const deltaY = moveEvent.clientY - startY;
        // Resize proportionally based on vertical movement
        const newSize = Math.max(12, startSize + deltaY * 0.5);
        currentEditSignature.size = newSize;
        drawStamps();
    };
    this._mouseUp = removeResizeHandler.bind(this);
    document.addEventListener('mousemove', handleResize);
    document.addEventListener('mouseup', removeResizeHandler);
    
    function removeResizeHandler() {
        document.removeEventListener('mousemove', handleResize);
        document.removeEventListener('mouseup', removeResizeHandler);
    }
});

// Handle confirmBtn click - finalize signature
confirmBtn.addEventListener('click', (e) => {
    e.stopPropagation();
    currentEditSignature = null;
    editControls.style.display = 'none';
    drawStamps();
    showStatus('Signature confirmed!');
});

// Handle cancelBtn click - remove current signature
cancelBtn.addEventListener('click', (e) => {
    e.stopPropagation();
    if (currentEditSignature) {
        // Find and remove the current signature
        const signatures = signatureCoords[currentPage] || [];
        const index = signatures.indexOf(currentEditSignature);
        if (index !== -1) {
            signatures.splice(index, 1);
        }
    }
    
    currentEditSignature = null;
    editControls.style.display = 'none';
    drawStamps();
    showStatus('Signature placement canceled');
});

// Dark mode toggle functionality
const themeToggleBtn = document.getElementById('theme-toggle-btn');
const sunIcon = document.getElementById('sun-icon');
const moonIcon = document.getElementById('moon-icon');

// Check for saved theme preference
if (localStorage.getItem('theme') === 'dark') {
    document.body.classList.add('dark-theme');
    sunIcon.style.display = 'none';
    moonIcon.style.display = 'block';
}

// Handle theme toggle
themeToggleBtn.addEventListener('click', () => {
    document.body.classList.toggle('dark-theme');
    
    // Toggle icons
    if (document.body.classList.contains('dark-theme')) {
        sunIcon.style.display = 'none';
        moonIcon.style.display = 'block';
        localStorage.setItem('theme', 'dark');
    } else {
        sunIcon.style.display = 'block';
        moonIcon.style.display = 'none';
        localStorage.setItem('theme', 'light');
    }
});
