document.addEventListener('DOMContentLoaded', () => {
    // --- DOM Element References ---
    const mainView = document.getElementById('main-view');
    const pdfViewer = document.getElementById('pdf-viewer');
    const openBtn = document.getElementById('open-btn');
    const lastOpenedList = document.getElementById('last-opened-list');
    const fileInput = document.getElementById('file-input');
    const canvas = document.getElementById('pdf-canvas');
    const ctx = canvas.getContext('2d');
    const closeBtn = document.getElementById('close-btn');
    const pageIndicator = document.getElementById('page-num-indicator');

    // --- PDF.js Setup ---
    pdfjsLib.GlobalWorkerOptions.workerSrc = `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.11.338/pdf.worker.min.js`;

    // --- State Management ---
    let pdfDoc = null;
    let currentPage = 1;
    let isRendering = false; // Prevents multiple renders/animations at once

    // --- IndexedDB for PDF Storage ---
    const DB_NAME = "PDFViewerAdvDB";
    const STORE_NAME = "recentPDFsStore";
    let db;

    function initDB() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(DB_NAME, 1);
            request.onupgradeneeded = event => {
                const db = event.target.result;
                if (!db.objectStoreNames.contains(STORE_NAME)) {
                    db.createObjectStore(STORE_NAME, { keyPath: 'id', autoIncrement: true });
                }
            };
            request.onsuccess = event => {
                db = event.target.result;
                resolve();
            };
            request.onerror = event => {
                console.error("IndexedDB error:", event.target.error);
                reject(event.target.error);
            };
        });
    }

    async function savePDF(pdfData, pdfName) {
        const transaction = db.transaction([STORE_NAME], "readwrite");
        const store = transaction.objectStore(STORE_NAME);
        const allRecords = await new Promise(resolve => store.getAll().onsuccess = e => resolve(e.target.result));
        
        // Remove existing entry with the same name
        const existingRecord = allRecords.find(record => record.name === pdfName);
        if (existingRecord) {
            store.delete(existingRecord.id);
        }

        // Add the new record
        store.add({ name: pdfName, data: pdfData, timestamp: new Date() });

        // Ensure only the 3 most recent files are kept
        if (allRecords.length >= 3) {
            allRecords.sort((a, b) => a.timestamp - b.timestamp); // Oldest first
            store.delete(allRecords[0].id);
        }
    }

    function getRecentPDFs() {
        return new Promise(resolve => {
            const transaction = db.transaction([STORE_NAME], "readonly");
            const store = transaction.objectStore(STORE_NAME);
            const request = store.getAll();
            request.onsuccess = e => {
                const sorted = e.target.result.sort((a, b) => b.timestamp - a.timestamp); // Newest first
                resolve(sorted);
            };
            request.onerror = () => resolve([]);
        });
    }

    // --- UI Initialization ---
    async function initUI() {
        await initDB();
        const recentPDFs = await getRecentPDFs();
        lastOpenedList.innerHTML = '';
        if (recentPDFs.length > 0) {
            recentPDFs.forEach(pdfInfo => {
                const link = document.createElement('a');
                link.href = '#';
                link.className = 'recent-file';
                link.textContent = pdfInfo.name;
                link.onclick = (e) => {
                    e.preventDefault();
                    displayPDF(pdfInfo.data, pdfInfo.name);
                };
                lastOpenedList.appendChild(link);
            });
        } else {
            lastOpenedList.innerHTML = '<p>No recent files.</p>';
        }
    }

    // --- PDF Rendering Logic ---
    async function renderPage(num, direction = null) {
        if (isRendering) return;
        isRendering = true;

        const page = await pdfDoc.getPage(num);
        const viewport = page.getViewport({ scale: window.devicePixelRatio * 1.5 }); // High-res rendering

        canvas.height = viewport.height;
        canvas.width = viewport.width;

        await page.render({ canvasContext: ctx, viewport: viewport }).promise;

        if (direction) {
            const animClassEnter = direction === 'next' ? 'slide-next-enter' : 'slide-prev-enter';
            canvas.classList.add(animClassEnter);
            canvas.onanimationend = () => {
                canvas.classList.remove(animClassEnter);
                canvas.onanimationend = null;
            };
        }
        
        currentPage = num;
        showPageIndicator();
        isRendering = false;
    }

    async function displayPDF(pdfData, pdfName) {
        try {
            const loadingTask = pdfjsLib.getDocument({ data: pdfData });
            pdfDoc = await loadingTask.promise;
            
            mainView.style.display = 'none';
            pdfViewer.style.display = 'flex';
            if (document.documentElement.requestFullscreen) {
                document.documentElement.requestFullscreen();
            }
            
            await renderPage(1);
        } catch (error) {
            console.error("Error loading PDF:", error);
            alert("Could not load the PDF file.");
            closeViewer();
        }
    }

    function showPageIndicator() {
        pageIndicator.textContent = `Page ${currentPage} / ${pdfDoc.numPages}`;
        pageIndicator.classList.add('fade-in-out');
        pageIndicator.onanimationend = () => {
            pageIndicator.classList.remove('fade-in-out');
            pageIndicator.onanimationend = null;
        };
    }

    function closeViewer() {
        pdfViewer.style.display = 'none';
        mainView.style.display = 'flex';
        if (document.fullscreenElement) {
            document.exitFullscreen();
        }
        pdfDoc = null;
        initUI(); // Refresh recent files list
    }

    // --- Touch & Swipe Navigation ---
    let touchStartY = 0;
    let touchStartX = 0;
    
    function handleTouchStart(e) {
        if (e.touches.length === 1) { // Only track single-finger swipes
            touchStartY = e.touches[0].clientY;
            touchStartX = e.touches[0].clientX;
        }
    }

    function handleTouchEnd(e) {
        if (e.changedTouches.length === 1) { // Ensure it's the end of a single touch
            const touchEndY = e.changedTouches[0].clientY;
            const deltaY = touchEndY - touchStartY;
            
            // Vertical swipe for page turning
            if (Math.abs(deltaY) > 50) { // Threshold for swipe
                if (deltaY > 0 && currentPage > 1) { // Swipe Down -> Previous Page
                    renderPage(currentPage - 1, 'prev');
                } else if (deltaY < 0 && currentPage < pdfDoc.numPages) { // Swipe Up -> Next Page
                    renderPage(currentPage + 1, 'next');
                }
            }
        }
    }

    // --- Event Listeners ---
    openBtn.addEventListener('click', () => fileInput.click());

    fileInput.addEventListener('change', async (event) => {
        const file = event.target.files[0];
        if (file && file.type === "application/pdf") {
            const arrayBuffer = await file.arrayBuffer();
            await savePDF(arrayBuffer, file.name);
            displayPDF(arrayBuffer, file.name);
        }
        fileInput.value = ''; // Reset input to allow opening the same file again
    });

    closeBtn.addEventListener('click', closeViewer);
    pdfViewer.addEventListener('touchstart', handleTouchStart, { passive: true });
    pdfViewer.addEventListener('touchend', handleTouchEnd, { passive: true });

    window.addEventListener('keydown', (e) => {
        if (!pdfDoc) return;
        if ((e.key === "ArrowUp" || e.key === "ArrowLeft") && currentPage > 1) {
            renderPage(currentPage - 1, 'prev');
        } else if ((e.key === "ArrowDown" || e.key === "ArrowRight") && currentPage < pdfDoc.numPages) {
            renderPage(currentPage + 1, 'next');
        }
    });

    // --- Initial Load ---
    initUI();
});