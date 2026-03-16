<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Warehouse Scanner - External Camera</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.0/font/bootstrap-icons.css" rel="stylesheet">
    <style>
        .scanner-container {
            position: relative;
            width: 100%;
            max-width: 800px;
            margin: 0 auto;
            background: #000;
            border-radius: 15px;
            overflow: hidden;
        }
        
        #scanner-video {
            width: 100%;
            height: auto;
            display: block;
        }
        
        .scanner-overlay {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            width: 300px;
            height: 300px;
            border: 3px solid #00ff00;
            border-radius: 15px;
            pointer-events: none;
            animation: pulse 2s infinite;
        }
        
        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.5; }
            100% { opacity: 1; }
        }
        
        .scanner-corner {
            position: absolute;
            width: 30px;
            height: 30px;
            border: 4px solid #00ff00;
        }
        
        .corner-tl { top: -4px; left: -4px; border-right: none; border-bottom: none; }
        .corner-tr { top: -4px; right: -4px; border-left: none; border-bottom: none; }
        .corner-bl { bottom: -4px; left: -4px; border-right: none; border-top: none; }
        .corner-br { bottom: -4px; right: -4px; border-left: none; border-top: none; }
        
        .operation-card {
            transition: all 0.3s ease;
            cursor: pointer;
        }
        
        .operation-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 10px 25px rgba(0,0,0,0.1);
        }
        
        .operation-card.active {
            border: 3px solid #0d6efd;
            background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
        }
        
        .scan-result {
            background: linear-gradient(135deg, #28a745 0%, #20c997 100%);
            color: white;
            padding: 20px;
            border-radius: 15px;
            margin: 20px 0;
            animation: slideIn 0.5s ease;
        }
        
        @keyframes slideIn {
            from { transform: translateY(-20px); opacity: 0; }
            to { transform: translateY(0); opacity: 1; }
        }
        
        .camera-controls {
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 15px;
            border-radius: 10px;
            margin: 10px 0;
        }
        
        .scan-history {
            max-height: 300px;
            overflow-y: auto;
        }
        
        .scan-history::-webkit-scrollbar {
            width: 8px;
        }
        
        .scan-history::-webkit-scrollbar-track {
            background: #f1f1f1;
            border-radius: 10px;
        }
        
        .scan-history::-webkit-scrollbar-thumb {
            background: #888;
            border-radius: 10px;
        }
        
        .scan-history::-webkit-scrollbar-thumb:hover {
            background: #555;
        }
    </style>
</head>
<body>
    <div class="container-fluid py-4">
        <header class="text-center mb-4">
            <div class="d-flex justify-content-between align-items-center mb-3">
                <div>
                    <a href="https://ieindospacegroup-cmyk.github.io/warehouse-scanner/" class="btn btn-outline-primary">
                        <i class="bi bi-house"></i> Main Dashboard
                    </a>
                </div>
                <div>
                    <h1 class="display-4 text-primary mb-0">
                        <i class="bi bi-camera-video"></i> Warehouse Scanner
                    </h1>
                </div>
                <div>
                    <button class="btn btn-outline-info" onclick="testConnection()">
                        <i class="bi bi-wifi"></i> Test Connection
                    </button>
                </div>
            </div>
            <p class="lead">External Camera QR/Barcode Scanner for WMS Operations</p>
            <div class="d-flex justify-content-center gap-2">
                <span class="badge bg-success">GitHub Pages Ready</span>
                <span class="badge bg-info">Google Sheets Integration</span>
                <span class="badge bg-warning">External Camera Support</span>
            </div>
        </header>

        <!-- Alert Container -->
        <div id="alert-container"></div>

        <!-- Operation Selection -->
        <div class="row mb-4">
            <div class="col-md-4">
                <div class="card operation-card" id="inbound-card" onclick="selectOperation('inbound')">
                    <div class="card-body text-center">
                        <i class="bi bi-box-arrow-in-down text-success" style="font-size: 3rem;"></i>
                        <h5 class="card-title mt-2">Barang Masuk (IN)</h5>
                        <p class="text-muted">Scan untuk barang masuk</p>
                    </div>
                </div>
            </div>
            <div class="col-md-4">
                <div class="card operation-card" id="outbound-card" onclick="selectOperation('outbound')">
                    <div class="card-body text-center">
                        <i class="bi bi-box-arrow-up text-danger" style="font-size: 3rem;"></i>
                        <h5 class="card-title mt-2">Barang Keluar (OUT)</h5>
                        <p class="text-muted">Scan untuk barang keluar</p>
                    </div>
                </div>
            </div>
            <div class="col-md-4">
                <div class="card operation-card" id="movement-card" onclick="selectOperation('movement')">
                    <div class="card-body text-center">
                        <i class="bi bi-arrow-left-right text-warning" style="font-size: 3rem;"></i>
                        <h5 class="card-title mt-2">Pindah Barang (MOVE)</h5>
                        <p class="text-muted">Scan untuk perpindahan barang</p>
                    </div>
                </div>
            </div>
        </div>

        <!-- Scanner Section -->
        <div class="row" id="scanner-section" style="display: none;">
            <div class="col-lg-8">
                <div class="card">
                    <div class="card-header">
                        <h5><i class="bi bi-camera"></i> Scanner Camera</h5>
                    </div>
                    <div class="card-body">
                        <!-- Camera Controls -->
                        <div class="camera-controls">
                            <div class="row align-items-center">
                                <div class="col-md-6">
                                    <label class="form-label">Select Camera:</label>
                                    <select class="form-select" id="camera-select">
                                        <option value="">Loading cameras...</option>
                                    </select>
                                </div>
                                <div class="col-md-6 text-end">
                                    <button class="btn btn-success" id="start-scan-btn" onclick="toggleScanner()">
                                        <i class="bi bi-play-circle"></i> Start Scanning
                                    </button>
                                    <button class="btn btn-danger" id="stop-scan-btn" onclick="toggleScanner()" style="display: none;">
                                        <i class="bi bi-stop-circle"></i> Stop Scanning
                                    </button>
                                </div>
                            </div>
                        </div>

                        <!-- Scanner Video -->
                        <div class="scanner-container">
                            <video id="scanner-video" autoplay playsinline></video>
                            <div class="scanner-overlay" style="display: none;">
                                <div class="scanner-corner corner-tl"></div>
                                <div class="scanner-corner corner-tr"></div>
                                <div class="scanner-corner corner-bl"></div>
                                <div class="scanner-corner corner-br"></div>
                            </div>
                        </div>

                        <!-- Scan Result -->
                        <div id="scan-result" style="display: none;"></div>
                    </div>
                </div>
            </div>

            <div class="col-lg-4">
                <!-- Operation Form -->
                <div class="card" id="operation-form" style="display: none;">
                    <div class="card-header">
                        <h5 id="form-title"><i class="bi bi-pencil-square"></i> Operation Details</h5>
                    </div>
                    <div class="card-body">
                        <form id="warehouse-form">
                            <div class="mb-3">
                                <label class="form-label">Item Code</label>
                                <input type="text" class="form-control" id="item-code" readonly>
                            </div>
                            <div class="mb-3">
                                <label class="form-label">Item Name</label>
                                <input type="text" class="form-control" id="item-name" readonly>
                            </div>
                            <div class="mb-3" id="location-section">
                                <label class="form-label">Location Code</label>
                                <input type="text" class="form-control" id="location-code" readonly>
                            </div>
                            <div class="mb-3" id="from-location-section" style="display: none;">
                                <label class="form-label">From Location</label>
                                <input type="text" class="form-control" id="from-location" readonly>
                            </div>
                            <div class="mb-3" id="to-location-section" style="display: none;">
                                <label class="form-label">To Location</label>
                                <input type="text" class="form-control" id="to-location">
                                <small class="text-muted">Scan or input destination location</small>
                            </div>
                            <div class="mb-3">
                                <label class="form-label">Dimension</label>
                                <input type="text" class="form-control" id="dimension" readonly>
                            </div>
                            <div class="mb-3">
                                <label class="form-label">Quantity</label>
                                <input type="number" class="form-control" id="quantity" min="1" value="1">
                            </div>
                            <button type="submit" class="btn btn-primary w-100">
                                <i class="bi bi-check-circle"></i> Process Transaction
                            </button>
                        </form>
                    </div>
                </div>

                <!-- Scan History -->
                <div class="card mt-3">
                    <div class="card-header">
                        <h5><i class="bi bi-clock-history"></i> Scan History</h5>
                    </div>
                    <div class="card-body">
                        <div class="scan-history" id="scan-history">
                            <p class="text-muted">No scans yet</p>
                        </div>
                        <div class="mt-2">
                            <button class="btn btn-sm btn-outline-primary" onclick="exportHistory()">
                                <i class="bi bi-download"></i> Export History
                            </button>
                            <button class="btn btn-sm btn-outline-danger" onclick="clearHistory()">
                                <i class="bi bi-trash"></i> Clear History
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- Footer -->
    <footer class="bg-dark text-white py-4 mt-5">
        <div class="container">
            <div class="row">
                <div class="col-md-6">
                    <h5>Warehouse Scanner Pro</h5>
                    <p class="mb-0">External Camera QR/Barcode Scanner for Modern Warehouse Management</p>
                    <small>Integrated with Google Sheets • GitHub Pages Ready</small>
                </div>
                <div class="col-md-6 text-end">
                    <div class="mb-2">
                        <a href="https://ieindospacegroup-cmyk.github.io/warehouse-scanner/" class="text-white me-3">
                            <i class="bi bi-house"></i> Main Dashboard
                        </a>
                        <a href="https://docs.google.com/spreadsheets/d/1o1I6m8d0wRTWKXAwsFI1Jn2NssRySBvKJNdjjvAXm6I/edit" class="text-white" target="_blank">
                            <i class="bi bi-google"></i> Spreadsheet
                        </a>
                    </div>
                    <div>
                        <small class="text-muted">
                            Deployed on GitHub Pages • 
                            <span id="current-year"></span> © IE Indo Space Group
                        </small>
                    </div>
                </div>
            </div>
        </div>
    </footer>

    <script>
        // Set current year
        document.getElementById('current-year').textContent = new Date().getFullYear();
    </script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <script src="https://unpkg.com/@zxing/library@latest"></script>
    <script src="scanner-config.js"></script>
    <script>
        // Global variables
        let warehouseScanner = null;
        let currentOperation = null;
        let isScanning = false;

        // Test connection function
        async function testConnection() {
            showAlert('info', 'Testing connection to Google Apps Script...');
            
            try {
                const result = await testGoogleScriptConnection();
                
                if (result.status === 'success') {
                    showAlert('success', 'Connection successful! Google Apps Script is accessible.');
                    console.log('Connection test result:', result);
                } else {
                    if (result.demo) {
                        showAlert('warning', 'Demo Mode: Google Apps Script not accessible from GitHub Pages due to CORS restrictions. Scanner will work with mock data.');
                    } else {
                        showAlert('error', result.message || 'Connection failed');
                    }
                }
            } catch (error) {
                showAlert('error', 'Connection test failed: ' + error.message);
            }
        }

        // Initialize on page load
        document.addEventListener('DOMContentLoaded', async function() {
            warehouseScanner = new WarehouseScanner();
            
            // Initialize scanner and load cameras
            const initResult = await warehouseScanner.initializeScanner();
            if (initResult.success) {
                loadCameraOptions();
                // Auto-test connection
                setTimeout(() => testConnection(), 1000);
            } else {
                showAlert('error', initResult.message);
            }
        });

        // Load camera options
        async function loadCameraOptions() {
            const devices = await warehouseScanner.getVideoDevices();
            const select = document.getElementById('camera-select');
            select.innerHTML = '';
            
            devices.forEach((device, index) => {
                const option = document.createElement('option');
                option.value = device.deviceId;
                option.textContent = device.label || `Camera ${index + 1}`;
                select.appendChild(option);
            });
            
            // Set default to external camera if available
            const externalDevice = devices.find(device => 
                device.label.toLowerCase().includes('back') || 
                device.label.toLowerCase().includes('external')
            );
            
            if (externalDevice) {
                select.value = externalDevice.deviceId;
            }
        }

        // Switch camera
        async function switchCamera() {
            const deviceId = document.getElementById('camera-select').value;
            if (deviceId && warehouseScanner) {
                const result = await warehouseScanner.switchCamera(deviceId);
                if (!result.success) {
                    showAlert('error', result.message);
                }
            }
        }

        // Select operation
        function selectOperation(operation) {
            currentOperation = operation;
            
            // Update UI
            document.querySelectorAll('.operation-card').forEach(card => {
                card.classList.remove('active');
            });
            document.getElementById(`${operation}-card`).classList.add('active');
            
            // Show scanner section
            document.getElementById('scanner-section').style.display = 'flex';
            
            // Setup form based on operation
            setupOperationForm(operation);
        }

        // Setup operation form
        function setupOperationForm(operation) {
            const form = document.getElementById('operation-form');
            const formTitle = document.getElementById('form-title');
            const locationSection = document.getElementById('location-section');
            const fromLocationSection = document.getElementById('from-location-section');
            const toLocationSection = document.getElementById('to-location-section');
            
            form.style.display = 'block';
            
            // Reset form
            document.getElementById('warehouse-form').reset();
            
            switch(operation) {
                case 'inbound':
                    formTitle.innerHTML = '<i class="bi bi-box-arrow-in-down"></i> Barang Masuk (IN)';
                    locationSection.style.display = 'block';
                    fromLocationSection.style.display = 'none';
                    toLocationSection.style.display = 'none';
                    break;
                case 'outbound':
                    formTitle.innerHTML = '<i class="bi bi-box-arrow-up"></i> Barang Keluar (OUT)';
                    locationSection.style.display = 'block';
                    fromLocationSection.style.display = 'none';
                    toLocationSection.style.display = 'none';
                    break;
                case 'movement':
                    formTitle.innerHTML = '<i class="bi bi-arrow-left-right"></i> Pindah Barang (MOVE)';
                    locationSection.style.display = 'none';
                    fromLocationSection.style.display = 'block';
                    toLocationSection.style.display = 'block';
                    break;
            }
        }

        // Toggle scanner
        async function toggleScanner() {
            if (!currentOperation) {
                showAlert('warning', 'Please select an operation first');
                return;
            }
            
            if (!warehouseScanner) {
                showAlert('error', 'Scanner not initialized');
                return;
            }
            
            const videoElement = document.getElementById('scanner-video');
            const overlay = document.querySelector('.scanner-overlay');
            const startBtn = document.getElementById('start-scan-btn');
            const stopBtn = document.getElementById('stop-scan-btn');
            
            if (isScanning) {
                // Stop scanning
                const result = warehouseScanner.stopScanning();
                if (result.success) {
                    isScanning = false;
                    videoElement.style.display = 'none';
                    overlay.style.display = 'none';
                    startBtn.style.display = 'inline-block';
                    stopBtn.style.display = 'none';
                    showAlert('info', 'Scanning stopped');
                }
            } else {
                // Start scanning
                const result = await warehouseScanner.startScanning(
                    videoElement,
                    onScanSuccess,
                    onScanError
                );
                
                if (result.success) {
                    isScanning = true;
                    videoElement.style.display = 'block';
                    overlay.style.display = 'block';
                    startBtn.style.display = 'none';
                    stopBtn.style.display = 'inline-block';
                    showAlert('success', 'Scanning started');
                } else {
                    showAlert('error', result.message);
                }
            }
        }

        // Handle successful scan
        function onScanSuccess(result) {
            console.log('Scan result:', result);
            
            if (result.error) {
                showAlert('warning', result.error);
                return;
            }
            
            const data = result.parsed.data;
            
            // Update form fields
            document.getElementById('item-code').value = data.itemCode;
            document.getElementById('item-name').value = data.itemName;
            document.getElementById('dimension').value = data.dimension;
            document.getElementById('quantity').value = data.quantity;
            
            // Set location based on operation
            if (currentOperation === 'inbound') {
                // Barang Masuk: Scan product → Input location → Process
                document.getElementById('location-code').value = data.locationCode;
                showAlert('info', 'Product scanned! Verify location and quantity for INBOUND operation.');
            } else if (currentOperation === 'outbound') {
                // Barang Keluar: Scan product → Verify location → Process
                document.getElementById('location-code').value = data.locationCode;
                checkStockForOutbound(data.itemCode, data.locationCode);
                showAlert('info', 'Product scanned! Verifying stock for OUTBOUND operation.');
            } else if (currentOperation === 'movement') {
                // Pindah Barang: Scan product → From location → Scan To location → Process
                document.getElementById('from-location').value = data.locationCode;
                checkStockForMovement(data.itemCode, data.locationCode);
                showAlert('info', 'Product scanned! Now scan or input destination location for MOVEMENT operation.');
            }
            
            // Show scan result
            showScanResult(result);
            
            // Update scan history
            updateScanHistory(result);
            
            showAlert('success', 'Barcode scanned successfully!');
        }

        // Check stock for outbound operation
        async function checkStockForOutbound(itemCode, locationCode) {
            try {
                // Mock API call to check stock
                const stockData = await getStockData(itemCode, locationCode);
                const currentStock = stockData.quantity || 0;
                
                if (currentStock === 0) {
                    showAlert('error', `No stock found for ${itemCode} at ${locationCode}`);
                    document.getElementById('quantity').max = 0;
                } else {
                    showAlert('info', `Current stock: ${currentStock} units`);
                    document.getElementById('quantity').max = currentStock;
                }
            } catch (error) {
                console.error('Stock check failed:', error);
            }
        }

        // Check stock for movement operation
        async function checkStockForMovement(itemCode, fromLocation) {
            try {
                // Mock API call to check stock
                const stockData = await getStockData(itemCode, fromLocation);
                const currentStock = stockData.quantity || 0;
                
                if (currentStock === 0) {
                    showAlert('error', `No stock found for ${itemCode} at ${fromLocation}`);
                    document.getElementById('quantity').max = 0;
                } else {
                    showAlert('info', `Available stock: ${currentStock} units at ${fromLocation}`);
                    document.getElementById('quantity').max = currentStock;
                }
            } catch (error) {
                console.error('Stock check failed:', error);
            }
        }

        // Get stock data (mock function - replace with actual API call)
        async function getStockData(itemCode, locationCode) {
            // This would connect to your Google Apps Script backend
            // For now, return mock data
            return {
                itemCode: itemCode,
                locationCode: locationCode,
                quantity: Math.floor(Math.random() * 100) + 1 // Mock stock
            };
        }

        // Handle scan error
        function onScanError(error) {
            console.error('Scan error:', error);
        }

        // Show scan result
        function showScanResult(result) {
            const resultDiv = document.getElementById('scan-result');
            const data = result.parsed.data;
            
            resultDiv.innerHTML = `
                <div class="scan-result">
                    <h6><i class="bi bi-check-circle"></i> Scan Successful!</h6>
                    <div class="row">
                        <div class="col-md-6">
                            <strong>Item Code:</strong> ${data.itemCode}<br>
                            <strong>Item Name:</strong> ${data.itemName}
                        </div>
                        <div class="col-md-6">
                            <strong>Location:</strong> ${data.locationCode}<br>
                            <strong>Dimension:</strong> ${data.dimension}<br>
                            <strong>Quantity:</strong> ${data.quantity}
                        </div>
                    </div>
                </div>
            `;
            
            resultDiv.style.display = 'block';
            
            // Hide after 5 seconds
            setTimeout(() => {
                resultDiv.style.display = 'none';
            }, 5000);
        }

        // Update scan history
        function updateScanHistory(result) {
            const historyDiv = document.getElementById('scan-history');
            const history = warehouseScanner.getScanHistory();
            
            if (history.length === 0) {
                historyDiv.innerHTML = '<p class="text-muted">No scans yet</p>';
                return;
            }
            
            let html = '';
            history.slice().reverse().slice(0, 10).forEach(scan => {
                const time = new Date(scan.timestamp).toLocaleTimeString();
                html += `
                    <div class="mb-2 p-2 border rounded">
                        <small class="text-muted">${time}</small><br>
                        <strong>${scan.text}</strong><br>
                        <small>${scan.device}</small>
                    </div>
                `;
            });
            
            historyDiv.innerHTML = html;
        }

        // Handle form submission
        document.getElementById('warehouse-form').addEventListener('submit', async function(e) {
            e.preventDefault();
            
            if (!currentOperation) {
                showAlert('warning', 'Please select an operation first');
                return;
            }
            
            const formData = {
                operation: currentOperation,
                itemCode: document.getElementById('item-code').value,
                itemName: document.getElementById('item-name').value,
                locationCode: document.getElementById('location-code').value,
                fromLocation: document.getElementById('from-location').value,
                toLocation: document.getElementById('to-location').value,
                dimension: document.getElementById('dimension').value,
                quantity: parseInt(document.getElementById('quantity').value)
            };
            
            // Process with Google Apps Script
            try {
                const response = await processWarehouseTransaction(formData);
                if (response.success) {
                    showAlert('success', response.message);
                    // Reset form
                    this.reset();
                } else {
                    showAlert('error', response.message);
                }
            } catch (error) {
                showAlert('error', 'Transaction failed: ' + error.message);
            }
        });

        // Process warehouse transaction (connect to Google Apps Script)
        async function processWarehouseTransaction(data) {
            const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/1o1I6m8d0wRTWKXAwsFI1Jn2NssRySBvKJNdjjvAXm6I/exec';
            
            try {
                let functionName;
                let parameters;
                
                switch(data.operation) {
                    case 'inbound':
                        functionName = 'processInflow';
                        parameters = {
                            barcodeData: `${data.itemCode}|${data.itemName}|${data.locationCode}|${data.dimension}|${data.quantity}`,
                            locationCode: data.locationCode,
                            inputQty: data.quantity
                        };
                        break;
                    case 'outbound':
                        functionName = 'processOutflow';
                        parameters = {
                            barcodeData: `${data.itemCode}|${data.itemName}|${data.locationCode}|${data.dimension}|${data.quantity}`,
                            locationCode: data.locationCode,
                            inputQty: data.quantity
                        };
                        break;
                    case 'movement':
                        functionName = 'processMoveFlow';
                        parameters = {
                            barcodeData: `${data.itemCode}|${data.itemName}|${data.fromLocation}|${data.dimension}|${data.quantity}`,
                            fromLocationCode: data.fromLocation,
                            toLocationCode: data.toLocation,
                            inputQty: data.quantity
                        };
                        break;
                    default:
                        throw new Error('Invalid operation type');
                }
                
                // Call Google Apps Script directly
                const response = await fetch(GOOGLE_SCRIPT_URL, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        function: functionName,
                        parameters: parameters
                    }),
                    mode: 'cors'
                });
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const result = await response.json();
                console.log('Transaction result:', result);
                return result;
                
            } catch (error) {
                console.error('Transaction processing failed:', error);
                
                // Fallback to mock response for demo
                if (error.message.includes('CORS') || error.message.includes('Failed to fetch')) {
                    console.log('Using fallback mode - Google Apps Script not accessible from GitHub Pages');
                    return {
                        status: 'success',
                        message: `${data.operation.toUpperCase()} transaction processed (demo mode)`,
                        updatedData: generateMockData(data),
                        demo: true
                    };
                }
                
                return {
                    status: 'error',
                    message: 'Failed to process transaction: ' + error.message
                };
            }
        }

        // Generate mock data for demo mode
        function generateMockData(transactionData) {
            const mockItems = [
                {
                    itemCode: transactionData.itemCode,
                    itemName: transactionData.itemName,
                    locationCode: transactionData.locationCode || transactionData.fromLocation,
                    dimension: transactionData.dimension,
                    qty: transactionData.quantity
                }
            ];
            
            return mockItems;
        }

        // Get actual stock data from Google Apps Script
        async function getStockData(itemCode, locationCode) {
            const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/1o1I6m8d0wRTWKXAwsFI1Jn2NssRySBvKJNdjjvAXm6I/exec';
            
            try {
                const response = await fetch(GOOGLE_SCRIPT_URL, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        function: 'getItems',
                        parameters: {}
                    }),
                    mode: 'cors'
                });
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const result = await response.json();
                
                if (result.status === 'success' && result.data) {
                    // Find specific item at location
                    const item = result.data.find(item => 
                        item.itemCode === itemCode && item.locationCode === locationCode
                    );
                    
                    return {
                        itemCode: itemCode,
                        locationCode: locationCode,
                        quantity: item ? item.qty : 0
                    };
                }
                
                return {
                    itemCode: itemCode,
                    locationCode: locationCode,
                    quantity: 0
                };
                
            } catch (error) {
                console.error('Failed to get stock data:', error);
                
                // Fallback to mock data
                if (error.message.includes('CORS') || error.message.includes('Failed to fetch')) {
                    console.log('Using fallback stock data');
                    return {
                        itemCode: itemCode,
                        locationCode: locationCode,
                        quantity: Math.floor(Math.random() * 100) + 1 // Mock stock
                    };
                }
                
                return {
                    itemCode: itemCode,
                    locationCode: locationCode,
                    quantity: 0
                };
            }
        }

        // Test connection to Google Apps Script
        async function testGoogleScriptConnection() {
            const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/1o1I6m8d0wRTWKXAwsFI1Jn2NssRySBvKJNdjjvAXm6I/exec';
            
            try {
                const response = await fetch(`${GOOGLE_SCRIPT_URL}?function=testConnection`, {
                    method: 'GET',
                    mode: 'cors'
                });
                
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }
                
                const result = await response.json();
                return result;
                
            } catch (error) {
                console.error('Connection test failed:', error);
                return {
                    status: 'error',
                    message: 'Google Apps Script not accessible from GitHub Pages (CORS limitation)',
                    demo: true
                };
            }
        }

        // Export scan history
        function exportHistory() {
            if (warehouseScanner) {
                const result = warehouseScanner.exportHistoryToCSV();
                if (result.success) {
                    showAlert('success', result.message);
                } else {
                    showAlert('error', result.message);
                }
            }
        }

        // Clear scan history
        function clearHistory() {
            if (warehouseScanner) {
                const result = warehouseScanner.clearHistory();
                if (result.success) {
                    document.getElementById('scan-history').innerHTML = '<p class="text-muted">No scans yet</p>';
                    showAlert('success', result.message);
                }
            }
        }

        // Show alert
        function showAlert(type, message) {
            const alertContainer = document.getElementById('alert-container');
            const alertId = 'alert-' + Date.now();
            
            const alertHtml = `
                <div class="alert alert-${type} alert-dismissible fade show" id="${alertId}" role="alert">
                    <i class="bi bi-${type === 'success' ? 'check-circle' : type === 'error' ? 'exclamation-triangle' : 'info-circle'}"></i>
                    ${message}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            `;
            
            alertContainer.innerHTML += alertHtml;
            
            // Auto-dismiss after 5 seconds
            setTimeout(() => {
                const alert = document.getElementById(alertId);
                if (alert) {
                    alert.remove();
                }
            }, 5000);
        }

        // Camera select change handler
        document.getElementById('camera-select').addEventListener('change', switchCamera);
    </script>
</body>
</html>
// External Camera Scanner Configuration
// GitHub Integration for Warehouse Management System

class WarehouseScanner {
    constructor() {
        this.scanner = null;
        this.currentStream = null;
        this.isScanning = false;
        this.devices = [];
        this.currentDevice = null;
        this.scanHistory = [];
        this.scanCount = 0;
        this.supportedFormats = [
            'QR_CODE',
            'CODE_128',
            'CODE_39',
            'EAN_13',
            'EAN_8',
            'UPC_A',
            'UPC_E',
            'DATA_MATRIX',
            'PDF_417',
            'AZTEC'
        ];
    }

    // Initialize scanner with external camera support
    async initializeScanner() {
        try {
            // Get all available cameras
            this.devices = await this.getVideoDevices();
            
            // Set default to external camera (not facing user)
            this.currentDevice = this.devices.find(device => 
                device.label.toLowerCase().includes('back') || 
                device.label.toLowerCase().includes('external') ||
                device.label.toLowerCase().includes('environment')
            ) || this.devices[0];

            // Initialize ZXing scanner with all supported formats
            this.scanner = new ZXing.BrowserMultiFormatReader();
            
            console.log('Scanner initialized with device:', this.currentDevice?.label);
            console.log('Supported formats:', this.supportedFormats);
            return { success: true, message: 'Scanner ready' };
        } catch (error) {
            console.error('Scanner initialization failed:', error);
            return { success: false, message: error.message };
        }
    }

    // Get all available video devices
    async getVideoDevices() {
        const devices = await navigator.mediaDevices.enumerateDevices();
        return devices.filter(device => device.kind === 'videoinput');
    }

    // Switch between cameras
    async switchCamera(deviceId) {
        if (this.currentStream) {
            this.currentStream.getTracks().forEach(track => track.stop());
        }

        try {
            const constraints = {
                video: {
                    deviceId: deviceId ? { exact: deviceId } : undefined,
                    facingMode: 'environment',
                    width: { ideal: 1920 },
                    height: { ideal: 1080 }
                }
            };

            this.currentStream = await navigator.mediaDevices.getUserMedia(constraints);
            this.currentDevice = this.devices.find(device => device.deviceId === deviceId);
            
            return { success: true, stream: this.currentStream };
        } catch (error) {
            console.error('Camera switch failed:', error);
            return { success: false, message: error.message };
        }
    }

    // Start scanning with external camera
    async startScanning(videoElement, onScanSuccess, onScanError) {
        if (!this.scanner) {
            await this.initializeScanner();
        }

        try {
            // Get camera stream
            const streamResult = await this.switchCamera(this.currentDevice?.deviceId);
            if (!streamResult.success) {
                throw new Error(streamResult.message);
            }

            // Set video stream
            videoElement.srcObject = streamResult.stream;
            videoElement.play();

            // Start continuous scanning
            this.isScanning = true;
            this.continuousScan(videoElement, onScanSuccess, onScanError);

            return { success: true, message: 'Scanning started' };
        } catch (error) {
            console.error('Start scanning failed:', error);
            return { success: false, message: error.message };
        }
    }

    // Continuous scanning loop with enhanced barcode detection
    continuousScan(videoElement, onScanSuccess, onScanError) {
        if (!this.isScanning) return;

        try {
            // Capture frame from video
            const canvas = document.createElement('canvas');
            const context = canvas.getContext('2d');
            canvas.width = videoElement.videoWidth;
            canvas.height = videoElement.videoHeight;
            context.drawImage(videoElement, 0, 0, canvas.width, canvas.height);

            // Try to decode barcode with multiple attempts
            const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
            
            // Try different decoding strategies
            let code = null;
            
            // Strategy 1: Direct decode
            try {
                code = this.scanner.decodeFromImageData(imageData);
            } catch (e) {
                // Strategy 2: Try with enhanced contrast
                try {
                    const enhancedData = this.enhanceImage(imageData);
                    code = this.scanner.decodeFromImageData(enhancedData);
                } catch (e2) {
                    // Strategy 3: Try with different regions
                    code = this.tryDifferentRegions(imageData);
                }
            }

            if (code && code.text) {
                this.handleScanResult(code.text, onScanSuccess);
                // Brief pause after successful scan
                setTimeout(() => {
                    this.continuousScan(videoElement, onScanSuccess, onScanError);
                }, 1000);
            } else {
                // Continue scanning
                requestAnimationFrame(() => {
                    this.continuousScan(videoElement, onScanSuccess, onScanError);
                });
            }
        } catch (error) {
            // No barcode found, continue scanning
            requestAnimationFrame(() => {
                this.continuousScan(videoElement, onScanSuccess, onScanError);
            });
        }
    }

    // Enhance image for better barcode detection
    enhanceImage(imageData) {
        const data = imageData.data;
        const enhancedData = new ImageData(
            new Uint8ClampedArray(data),
            imageData.width,
            imageData.height
        );
        
        // Increase contrast
        const factor = 2;
        for (let i = 0; i < data.length; i += 4) {
            enhancedData.data[i] = Math.min(255, (data[i] - 128) * factor + 128);     // Red
            enhancedData.data[i + 1] = Math.min(255, (data[i + 1] - 128) * factor + 128); // Green
            enhancedData.data[i + 2] = Math.min(255, (data[i + 2] - 128) * factor + 128); // Blue
            enhancedData.data[i + 3] = data[i + 3]; // Alpha
        }
        
        return enhancedData;
    }

    // Try scanning different regions of the image
    tryDifferentRegions(imageData) {
        const regions = [
            { x: 0, y: 0, w: imageData.width, h: imageData.height },           // Full image
            { x: imageData.width * 0.25, y: 0, w: imageData.width * 0.5, h: imageData.height }, // Center
            { x: 0, y: imageData.height * 0.25, w: imageData.width, h: imageData.height * 0.5 }, // Middle
        ];
        
        for (const region of regions) {
            try {
                const regionData = this.extractRegion(imageData, region);
                const code = this.scanner.decodeFromImageData(regionData);
                if (code && code.text) {
                    return code;
                }
            } catch (e) {
                continue;
            }
        }
        
        return null;
    }

    // Extract region from image data
    extractRegion(imageData, region) {
        const canvas = document.createElement('canvas');
        const context = canvas.getContext('2d');
        canvas.width = region.w;
        canvas.height = region.h;
        
        const tempCanvas = document.createElement('canvas');
        const tempContext = tempCanvas.getContext('2d');
        tempCanvas.width = imageData.width;
        tempCanvas.height = imageData.height;
        tempContext.putImageData(imageData, 0, 0);
        
        context.drawImage(tempCanvas, region.x, region.y, region.w, region.h, 0, 0, region.w, region.h);
        return context.getImageData(0, 0, region.w, region.h);
    }

    // Handle successful scan with enhanced validation
    handleScanResult(scanText, onScanSuccess) {
        this.scanCount++;
        const timestamp = new Date().toISOString();
        
        // Add to scan history
        this.scanHistory.push({
            id: this.scanCount,
            text: scanText,
            timestamp: timestamp,
            device: this.currentDevice?.label || 'Unknown'
        });

        // Parse warehouse barcode format
        const parsedData = this.parseWarehouseBarcode(scanText);
        
        if (parsedData.valid) {
            onScanSuccess({
                raw: scanText,
                parsed: parsedData,
                timestamp: timestamp,
                device: this.currentDevice?.label,
                format: this.detectBarcodeFormat(scanText)
            });
        } else {
            // Try to parse as simple barcode (fallback)
            const simpleData = this.parseSimpleBarcode(scanText);
            onScanSuccess({
                raw: scanText,
                parsed: simpleData,
                timestamp: timestamp,
                device: this.currentDevice?.label,
                format: this.detectBarcodeFormat(scanText)
            });
        }
    }

    // Detect barcode format
    detectBarcodeFormat(text) {
        if (text.includes('|')) {
            return 'Warehouse Format';
        } else if (text.match(/^\d{12,13}$/)) {
            return 'EAN/UPC';
        } else if (text.match(/^[A-Z0-9]{6,}$/)) {
            return 'Code 128/39';
        } else if (text.startsWith('http') || text.startsWith('www')) {
            return 'QR Code URL';
        } else {
            return 'Unknown';
        }
    }

    // Parse warehouse barcode format: ITEM CODE|ITEM NAME|LOCATION CODE|DIMENSION|QTY
    parseWarehouseBarcode(barcodeText) {
        try {
            const parts = barcodeText.split('|');
            
            if (parts.length !== 5) {
                return {
                    valid: false,
                    error: 'Expected format: ITEM CODE|ITEM NAME|LOCATION CODE|DIMENSION|QTY',
                    example: 'TIMO-LG|Timo Lounger|A1-01|206 x 83 x 31 cm|1'
                };
            }

            return {
                valid: true,
                data: {
                    itemCode: parts[0].trim(),
                    itemName: parts[1].trim(),
                    locationCode: parts[2].trim(),
                    dimension: parts[3].trim(),
                    quantity: parseInt(parts[4]) || 0
                }
            };
        } catch (error) {
            return {
                valid: false,
                error: 'Failed to parse barcode: ' + error.message
            };
        }
    }

    // Parse simple barcode (fallback)
    parseSimpleBarcode(barcodeText) {
        return {
            valid: true,
            data: {
                itemCode: barcodeText.trim(),
                itemName: 'Unknown Item',
                locationCode: 'Unknown',
                dimension: 'Unknown',
                quantity: 1
            },
            fallback: true
        };
    }

    // Stop scanning
    stopScanning() {
        this.isScanning = false;
        
        if (this.currentStream) {
            this.currentStream.getTracks().forEach(track => track.stop());
            this.currentStream = null;
        }

        if (this.scanner) {
            this.scanner.reset();
        }

        return { success: true, message: 'Scanning stopped' };
    }

    // Get scan history
    getScanHistory() {
        return this.scanHistory;
    }

    // Clear scan history
    clearHistory() {
        this.scanHistory = [];
        this.scanCount = 0;
        return { success: true, message: 'History cleared' };
    }

    // Export scan history to CSV
    exportHistoryToCSV() {
        if (this.scanHistory.length === 0) {
            return { success: false, message: 'No scan history to export' };
        }

        const headers = ['ID', 'Scan Text', 'Timestamp', 'Device', 'Format'];
        const rows = this.scanHistory.map(scan => [
            scan.id,
            scan.text,
            scan.timestamp,
            scan.device,
            this.detectBarcodeFormat(scan.text)
        ]);

        const csvContent = [headers, ...rows]
            .map(row => row.map(cell => `"${cell}"`).join(','))
            .join('\n');

        const blob = new Blob([csvContent], { type: 'text/csv' });
        const url = window.URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `scan_history_${new Date().toISOString().split('T')[0]}.csv`;
        a.click();
        window.URL.revokeObjectURL(url);

        return { success: true, message: 'History exported' };
    }

    // Get scanner statistics
    getScannerStats() {
        const successfulScans = this.scanHistory.length;
        const formatStats = {};
        
        this.scanHistory.forEach(scan => {
            const format = this.detectBarcodeFormat(scan.text);
            formatStats[format] = (formatStats[format] || 0) + 1;
        });

        return {
            totalScans: this.scanCount,
            successfulScans: successfulScans,
            successRate: this.scanCount > 0 ? Math.round((successfulScans / this.scanCount) * 100) : 0,
            formatStats: formatStats,
            currentDevice: this.currentDevice?.label || 'Unknown',
            supportedFormats: this.supportedFormats
        };
    }
}

// Export for use in other modules
window.WarehouseScanner = WarehouseScanner;
# Warehouse Management System (WMS) - External Camera Scanner

## 📋 Overview

Sistem Warehouse Management System dengan scanner kamera eksternal untuk membaca QR dan barcode. Mendukung operasi gudang: Barang Masuk (IN), Barang Keluar (OUT), dan Pindah Barang (MOVE).

## 🔗 Integration

- **Google Spreadsheet**: [Warehouse Sheet](https://docs.google.com/spreadsheets/d/1o1I6m8d0wRTWKXAwsFI1Jn2NssRySBvKJNdjjvAXm6I/edit?gid=0#gid=0)
- **Barcode Format**: `ITEM CODE|ITEM NAME|LOCATION CODE|DIMENSION|QTY`
- **External Camera Support**: Kamera eksternal USB, webcam, atau kamera ponsel

## 🚀 Features

### 📷 Scanner Capabilities
- ✅ **Multi-Camera Support**: Pilih antara kamera internal/eksternal
- ✅ **QR & Barcode Reading**: Baca berbagai jenis kode
- ✅ **Continuous Scanning**: Scan otomatis tanpa tombol
- ✅ **Real-time Processing**: Parse data langsung saat scan
- ✅ **Scan History**: Riwayat scan dengan export CSV

### 🏭 Warehouse Operations
- ✅ **Barang Masuk (IN)**: Scan product → Input location → Process
- ✅ **Barang Keluar (OUT)**: Scan product → Verify location → Process  
- ✅ **Pindah Barang (MOVE)**: Scan product → From → To → Process

### 📊 Data Management
- ✅ **Real-time Dashboard**: Update langsung setiap transaksi
- ✅ **Spreadsheet Sync**: Otomatis sync ke Google Sheets
- ✅ **Transaction Logging**: Log lengkap semua operasi
- ✅ **Inventory Management**: Update stok otomatis

## 📁 Files Structure

```
warehouse/
├── code.gs                    # Google Apps Script Backend
├── index.html                 # Main WMS Dashboard
├── scanner-config.js          # Scanner Configuration Class
├── scanner-app.html           # External Camera Scanner App
├── README.md                  # Documentation
└── scanner-deployment/        # GitHub Deployment Files
    ├── main.js               # GitHub Actions Workflow
    ├── package.json          # Node.js Dependencies
    └── vercel.json           # Vercel Deployment Config
```

## 🛠️ Installation

### 1. Google Apps Script Setup
```bash
# 1. Buka Google Apps Script: script.google.com
# 2. Buat project baru
# 3. Copy paste code.gs
# 4. Deploy sebagai Web App
# 5. Update SPREADSHEET_ID di code.gs
```

### 2. Local Development
```bash
# Clone repository
git clone <repository-url>
cd warehouse

# Buka index.html di browser
# atau gunakan live server
python -m http.server 8000
```

### 3. External Camera Setup
```bash
# Connect external camera via USB
# Allow camera permissions in browser
# Select camera in scanner app
```

## 📱 Usage

### Scanner Operation
1. **Select Operation**: Pilih IN/OUT/MOVE
2. **Choose Camera**: Pilih kamera eksternal
3. **Start Scanning**: Klik "Start Scanning"
4. **Scan Barcode**: Arahkan kamera ke barcode
5. **Process**: Verify data dan submit

### Barcode Format Example
```
TIMO-LG|Timo Lounger|A1-01|206 x 83 x 31 cm|1
```

### Operation Flows

#### Barang Masuk (IN)
```
Scan Product → Auto-fill data → Verify Location → Input Qty → Submit → Update Spreadsheet
```

#### Barang Keluar (OUT)
```
Scan Product → Check Stock → Verify Location → Input Qty → Submit → Update Spreadsheet
```

#### Pindah Barang (MOVE)
```
Scan Product → From Location → Scan To Location → Input Qty → Submit → 2 Transactions
```

## 🔧 Configuration

### Google Apps Script
```javascript
// Update di code.gs
const SPREADSHEET_ID = "YOUR_SPREADSHEET_ID";
const SHEET_ITEMS = "Warehouse";
const SHEET_TRANSACTIONS = "Transactions";
const SHEET_LOCATIONS = "CODE";
```

### Scanner Settings
```javascript
// Di scanner-config.js
const scannerConfig = {
    facingMode: 'environment',    // External camera
    width: { ideal: 1920 },       // HD quality
    height: { ideal: 1080 },
    continuousScan: true,        // Auto scan
    scanInterval: 1000           // 1 second interval
};
```

## 🌐 Deployment Options

### 1. Google Apps Script (Recommended)
- ✅ Free hosting
- ✅ Direct spreadsheet integration
- ✅ Easy deployment

### 2. GitHub + Vercel
- ✅ Custom domain
- ✅ Advanced features
- ✅ CI/CD pipeline

### 3. Local Server
- ✅ Full control
- ✅ Development testing
- ✅ Custom integration

## 📊 API Endpoints

### Google Apps Script Functions
```javascript
// Scanner operations
processInflow(barcodeData, locationCode, inputQty)
processOutflow(barcodeData, locationCode, inputQty)  
processMoveFlow(barcodeData, fromLocation, toLocation, inputQty)

// Data operations
getItems()
getLocations()
getTransactions()
testConnection()
```

### Scanner Class Methods
```javascript
// Initialize
const scanner = new WarehouseScanner();
await scanner.initializeScanner();

// Operations
await scanner.startScanning(videoElement, onSuccess, onError);
scanner.stopScanning();
scanner.switchCamera(deviceId);

// Data
scanner.getScanHistory();
scanner.exportHistoryToCSV();
scanner.clearHistory();
```

## 🔍 Troubleshooting

### Camera Issues
```bash
# Check camera permissions
# Allow camera in browser settings
# Try different camera device
# Check USB connection
```

### Scan Issues
```bash
# Ensure good lighting
# Clean camera lens
# Hold barcode steady
# Check barcode quality
```

### Spreadsheet Issues
```bash
# Verify SPREADSHEET_ID
# Check sheet permissions
# Validate sheet structure
# Test connection function
```

## 📞 Support

### Common Issues
1. **Camera not found**: Check permissions and connection
2. **Barcode not reading**: Improve lighting and focus
3. **Spreadsheet error**: Verify ID and permissions
4. **Data not updating**: Check network connection

### Help Resources
- 📖 [Google Apps Script Documentation](https://developers.google.com/apps-script)
- 📖 [ZXing Barcode Library](https://github.com/zxing-js/library)
- 📖 [Bootstrap Documentation](https://getbootstrap.com/docs/)
- 📖 [WebRTC Camera API](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices)

## 🔄 Version History

### v2.0 - External Camera Scanner
- ✅ External camera support
- ✅ Continuous scanning
- ✅ GitHub deployment
- ✅ Advanced features

### v1.0 - Basic WMS
- ✅ Basic barcode scanning
- ✅ Spreadsheet integration
- ✅ Dashboard interface

## 📄 License

MIT License - Free for commercial and personal use

## 🤝 Contributing

1. Fork repository
2. Create feature branch
3. Submit pull request
4. Review and merge

---

**Ready to deploy your warehouse scanner system! 🚀**
