[Uploading Index.html…]()
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ระบบค้นหาข้อมูล Excel (Hybrid Cloud)</title>
    
    <!-- Tailwind CSS & Icons -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    
    <!-- SheetJS (Excel) -->
    <script src="https://cdn.sheetjs.com/xlsx-latest/package/dist/xlsx.full.min.js"></script>
    
    <!-- Firebase SDK (Version 9 - Modular) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-app.js";
        import { getFirestore, doc, getDoc, setDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore.js";
        import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-auth.js";
        
        // --- ส่วนการตั้งค่า Firebase ---
        const defaultFirebaseConfig = {
            apiKey: "AIzaSyB4KOYrlUtYRAnv3fFeYPtidOp6MnbbIts",
            authDomain: "nyb2-34c1f.firebaseapp.com",
            databaseURL: "https://nyb2-34c1f-default-rtdb.firebaseio.com",
            projectId: "nyb2-34c1f",
            storageBucket: "nyb2-34c1f.firebasestorage.app",
            messagingSenderId: "1004488431337",
            appId: "1:1004488431337:web:e4fd5bc0a529bb2212c278",
            measurementId: "G-5KBBGTGVB9"
        };

        window.initFirebaseApp = async (config = defaultFirebaseConfig) => {
            try {
                const app = initializeApp(config);
                const auth = getAuth(app);
                
                try {
                    await signInAnonymously(auth);
                    console.log("✅ Signed in anonymously");
                } catch (authError) {
                    console.warn("⚠️ Auth Warning:", authError.message);
                }

                window.db = getFirestore(app);
                window.dbDocRef = doc(window.db, "app_data", "master_file");
                return true;
            } catch (e) {
                console.error("❌ Firebase Init Error:", e);
                return false;
            }
        };
        
        window.saveToFirestore = async (jsonData) => {
            if (!window.db) return false;
            try {
                await setDoc(window.dbDocRef, { 
                    updatedAt: new Date().toISOString(),
                    data: JSON.stringify(jsonData) 
                });
                return true;
            } catch (e) {
                console.error("❌ Save Error:", e.code);
                handleFirebaseError(e);
                return false;
            }
        };
        
        window.deleteFromFirestore = async () => {
            if (!window.db) return false;
            try {
                await deleteDoc(window.dbDocRef);
                return true;
            } catch (e) {
                console.error("❌ Delete Error:", e);
                handleFirebaseError(e);
                return false;
            }
        };

        window.loadFromFirestore = async () => {
            if (!window.db) return null;
            try {
                const docSnap = await getDoc(window.dbDocRef);
                if (docSnap.exists()) {
                    const content = docSnap.data();
                    return {
                        data: JSON.parse(content.data),
                        updatedAt: content.updatedAt
                    };
                }
                return { data: [], updatedAt: null }; 
            } catch (e) {
                if (e.code === 'permission-denied' || e.message.includes('permission')) {
                    handleFirebaseError(e);
                    return null; 
                }
                return null;
            }
        };

        function handleFirebaseError(error) {
            if (document.getElementById('rules-help-modal').classList.contains('opacity-0')) {
                if (error.code === 'permission-denied' || error.message.includes('Missing or insufficient permissions')) {
                    toggleModal('rules-help-modal');
                    updateStatusUI(false, "ติดสิทธิ์การเข้าถึง (Permission)");
                }
            }
        }
    </script>

    <!-- Font -->
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Sarabun', sans-serif; background-color: #f0f2f5; }
        .animate-fade-in { animation: fadeIn 0.4s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        .modal { transition: opacity 0.25s ease; }
        body.modal-active { overflow-x: hidden; overflow-y: visible !important; }
        
        /* New Admin Widget Styles */
        .admin-panel-enter {
            transform: scale(0.95);
            opacity: 0;
            pointer-events: none;
        }
        .admin-panel-active {
            transform: scale(1);
            opacity: 1;
            pointer-events: auto;
        }
    </style>
</head>
<body class="min-h-screen p-4 flex flex-col items-center relative">

    <!-- ADMIN WIDGET (Top Right Fixed) -->
    <div id="admin-widget" class="fixed top-4 right-4 z-50 flex flex-col items-end">
        
        <!-- Toggle Button -->
        <button onclick="toggleAdminPanel()" class="bg-slate-800 text-white w-12 h-12 rounded-full shadow-lg hover:bg-slate-700 transition-all border-2 border-slate-600 group relative flex items-center justify-center z-50">
            <i class="fa-solid fa-user-shield text-lg text-yellow-400"></i>
            <!-- Tooltip -->
            <span class="absolute right-full mr-3 top-1/2 -translate-y-1/2 bg-slate-900 text-white text-xs px-2 py-1 rounded opacity-0 group-hover:opacity-100 whitespace-nowrap transition-opacity shadow-sm pointer-events-none">
                Admin Zone
            </span>
        </button>

        <!-- The Admin Panel -->
        <div id="admin-panel" class="absolute top-14 right-0 w-72 bg-slate-800 text-white rounded-xl shadow-2xl border border-slate-700 overflow-hidden transition-all duration-300 origin-top-right admin-panel-enter">
            <!-- Header -->
            <div class="px-5 py-3 bg-slate-900/50 border-b border-slate-700 flex justify-between items-center">
                <span class="text-sm font-bold text-slate-200">Admin Control</span>
                <button id="admin-logout-btn" onclick="logoutAdmin()" class="hidden text-[10px] bg-slate-700 hover:bg-slate-600 px-2 py-0.5 rounded text-red-300 transition-colors">
                    Logout
                </button>
            </div>

            <div class="p-5">
                <!-- LOCKED STATE -->
                <div id="admin-locked" class="text-center">
                    <div class="w-12 h-12 bg-slate-700 rounded-full flex items-center justify-center mx-auto mb-3">
                        <i class="fa-solid fa-lock text-slate-400 text-lg"></i>
                    </div>
                    <p class="text-xs text-slate-400 mb-4">กรุณายืนยันตัวตนเพื่อจัดการข้อมูล</p>
                    <button onclick="requestAdminAccess()" class="w-full bg-blue-600 hover:bg-blue-500 text-white font-bold py-2 rounded-lg transition-all text-sm shadow-md">
                        เข้าสู่ระบบ
                    </button>
                </div>

                <!-- UNLOCKED STATE -->
                <div id="admin-unlocked" class="hidden">
                    <!-- Stats -->
                    <div class="grid grid-cols-2 gap-2 mb-4">
                        <div class="bg-slate-700/50 p-2 rounded text-center">
                            <div class="text-[10px] text-slate-400">Records</div>
                            <div class="text-lg font-bold text-white" id="admin-record-count">-</div>
                        </div>
                        <div class="bg-slate-700/50 p-2 rounded text-center">
                            <div class="text-[10px] text-slate-400">Updated</div>
                            <div class="text-[10px] font-bold text-green-400 mt-1 truncate" id="admin-last-update">-</div>
                        </div>
                    </div>

                    <!-- Actions -->
                    <div class="space-y-3">
                        <div class="relative">
                            <input type="file" id="master-db-input" accept=".xlsx, .xls" class="hidden" onchange="handleFileSelect(this)">
                            <label for="master-db-input" class="cursor-pointer w-full bg-slate-700 hover:bg-slate-600 text-slate-200 text-xs py-2 px-3 rounded-lg border border-slate-600 border-dashed flex items-center justify-center gap-2 transition-colors">
                                <i class="fa-solid fa-file-excel text-green-500"></i>
                                <span id="master-file-name" class="truncate max-w-[150px]">เลือกไฟล์ Excel...</span>
                            </label>
                        </div>
                        
                        <button onclick="triggerDbUpload()" id="btn-upload-db" class="w-full bg-purple-600 hover:bg-purple-500 text-white font-bold py-2 rounded-lg shadow-md transition-colors flex justify-center items-center gap-2 text-xs opacity-50 cursor-not-allowed" disabled>
                            <i class="fa-solid fa-cloud-arrow-up"></i> อัปโหลด
                        </button>

                        <div class="pt-2 mt-2 border-t border-slate-700/50">
                            <button onclick="deleteDatabase()" class="w-full text-red-400 hover:bg-red-500/10 py-1.5 rounded transition-colors text-xs flex justify-center items-center gap-1">
                                <i class="fa-solid fa-trash-can"></i> ล้างข้อมูล (Reset)
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- MAIN UI -->
    <div class="w-full max-w-md flex justify-between items-center mb-4 px-2 mt-8 md:mt-0">
        <div id="connection-status" class="flex items-center gap-2 text-xs font-bold text-gray-400">
            <div class="w-2 h-2 rounded-full bg-orange-400 animate-pulse"></div> Connecting...
        </div>
        <button onclick="toggleModal('settings-modal')" class="text-gray-400 hover:text-blue-600 transition-colors text-xs">
            <i class="fa-solid fa-gear"></i> ตั้งค่า
        </button>
    </div>

    <div class="bg-white rounded-[20px] shadow-xl w-full max-w-md p-6 space-y-4 animate-fade-in border border-gray-100 z-10 relative mb-6">
        <div class="flex items-center gap-2 mb-2">
            <i class="fa-solid fa-magnifying-glass text-blue-600 text-xl"></i>
            <h2 class="text-xl font-bold text-gray-800">ค้นหาข้อมูล</h2>
        </div>
        
        <textarea id="search-textarea" rows="5" 
            class="w-full border border-blue-200 rounded-xl p-3 text-sm focus:ring-2 focus:ring-blue-500 outline-none resize-none bg-white text-gray-700 placeholder-gray-300 shadow-inner"
            placeholder="ใส่ข้อมูลที่ต้องการค้นหา... (บรรทัดละ 1 รายการ)"></textarea>

        <div class="flex gap-2">
            <button onclick="performBatchSearch()" class="flex-1 bg-gradient-to-r from-blue-600 to-blue-700 text-white font-bold py-3 px-4 rounded-xl shadow-lg hover:from-blue-700 hover:to-blue-800 transition-all transform active:scale-95 flex justify-center items-center gap-2">
                <i class="fa-solid fa-magnifying-glass"></i> ค้นหา
            </button>
            <label class="cursor-pointer bg-gray-100 text-gray-600 border border-gray-200 font-bold py-3 px-4 rounded-xl hover:bg-gray-200 transition-all active:scale-95 flex items-center justify-center" title="โหมด Local">
                <i class="fa-solid fa-folder-open"></i>
                <input type="file" id="local-file-input" accept=".xlsx, .xls" class="hidden" onchange="useLocalFile(this)">
            </label>
        </div>
        
        <p id="data-source-info" class="text-center text-xs text-gray-400 mt-2 font-light">
            <i class="fa-solid fa-circle-notch fa-spin"></i> กำลังเชื่อมต่อฐานข้อมูล...
        </p>
    </div>

    <!-- RESULT SECTION -->
    <div id="result-section" class="w-full max-w-5xl mt-2 hidden animate-fade-in mb-10">
        <div class="bg-white rounded-xl shadow-lg overflow-hidden border border-gray-200">
            <div class="bg-gray-50 px-6 py-4 border-b border-gray-200 flex justify-between items-center">
                <div class="flex items-center gap-4">
                    <h3 class="font-bold text-gray-700">ผลลัพธ์ <span id="found-count" class="bg-blue-100 text-blue-800 text-xs px-2 py-1 rounded-full ml-2">0</span></h3>
                    <!-- Export Button -->
                    <button onclick="exportToExcel()" class="bg-green-600 hover:bg-green-500 text-white text-xs px-3 py-1.5 rounded-lg flex items-center gap-1 transition-colors shadow-sm font-bold">
                        <i class="fa-solid fa-file-export"></i> Export Excel
                    </button>
                </div>
                <button onclick="document.getElementById('result-section').classList.add('hidden')" class="text-gray-400 hover:text-gray-600"><i class="fa-solid fa-xmark"></i></button>
            </div>
            <div class="overflow-x-auto max-h-[500px]">
                <table class="min-w-full divide-y divide-gray-200">
                    <thead class="bg-gray-100 sticky top-0"><tr id="table-header"></tr></thead>
                    <tbody id="table-body" class="bg-white divide-y divide-gray-200"></tbody>
                </table>
            </div>
        </div>
    </div>

    <!-- MODALS (Password, Rules, Settings) -->
    <div id="password-modal" class="modal opacity-0 pointer-events-none fixed w-full h-full top-0 left-0 flex items-center justify-center z-[60]">
        <div class="modal-overlay absolute w-full h-full bg-gray-900 opacity-50"></div>
        <div class="modal-container bg-white w-11/12 md:max-w-xs mx-auto rounded shadow-lg z-50 overflow-y-auto">
            <div class="modal-content py-4 text-left px-6">
                <div class="flex justify-between items-center pb-3">
                    <p class="text-lg font-bold">ยืนยันรหัสผ่าน</p>
                    <div class="cursor-pointer" onclick="toggleModal('password-modal')"><i class="fa-solid fa-xmark"></i></div>
                </div>
                <input type="password" id="admin-password-input" class="w-full border p-2 rounded-lg mb-4 text-center tracking-widest" placeholder="Password">
                <div class="flex justify-end pt-2">
                    <button onclick="checkAdminPassword()" class="w-full px-4 py-2 bg-slate-800 text-white rounded-lg hover:bg-slate-700 font-bold shadow-md">ยืนยัน</button>
                </div>
            </div>
        </div>
    </div>

    <div id="rules-help-modal" class="modal opacity-0 pointer-events-none fixed w-full h-full top-0 left-0 flex items-center justify-center z-[60]">
        <div class="modal-overlay absolute w-full h-full bg-gray-900 opacity-70"></div>
        <div class="modal-container bg-white w-11/12 md:max-w-lg mx-auto rounded-lg shadow-2xl z-50 overflow-y-auto">
            <div class="modal-content py-6 text-left px-6">
                <div class="flex justify-between items-center pb-3 border-b mb-4">
                    <p class="text-xl font-bold text-red-600"><i class="fa-solid fa-triangle-exclamation"></i> ติดสิทธิ์การเข้าถึง</p>
                    <div class="cursor-pointer" onclick="toggleModal('rules-help-modal')"><i class="fa-solid fa-xmark text-xl"></i></div>
                </div>
                <p class="text-sm text-gray-700 mb-4">แก้ไข Rules ใน Firebase Console:</p>
                <div class="relative bg-gray-800 text-white p-3 rounded-lg text-xs font-mono mb-4">
                    <pre id="rules-code">rules_version = '2'; service cloud.firestore { match /databases/{database}/documents { match /{document=**} { allow read, write: if true; } } }</pre>
                    <button onclick="copyRules()" class="absolute top-2 right-2 bg-gray-600 hover:bg-gray-500 text-white px-2 py-1 rounded text-xs">Copy</button>
                </div>
                <div class="flex justify-end pt-4"><button onclick="toggleModal('rules-help-modal')" class="px-6 py-2 bg-gray-200 text-gray-700 rounded hover:bg-gray-300">ปิด</button></div>
            </div>
        </div>
    </div>

    <div id="settings-modal" class="modal opacity-0 pointer-events-none fixed w-full h-full top-0 left-0 flex items-center justify-center z-[60]">
        <div class="modal-overlay absolute w-full h-full bg-gray-900 opacity-50"></div>
        <div class="modal-container bg-white w-11/12 md:max-w-md mx-auto rounded shadow-lg z-50">
            <div class="modal-content py-4 text-left px-6">
                <div class="flex justify-between items-center pb-3"><p class="text-2xl font-bold">Config</p><div class="cursor-pointer" onclick="toggleModal('settings-modal')"><i class="fa-solid fa-xmark"></i></div></div>
                <textarea id="firebase-config-input" class="w-full h-40 border p-2 rounded text-xs font-mono bg-gray-50"></textarea>
                <div class="flex justify-end pt-2"><button onclick="saveSettings()" class="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700">บันทึก</button></div>
            </div>
        </div>
    </div>

    <div id="toast" class="fixed bottom-5 left-1/2 transform -translate-x-1/2 bg-gray-800 text-white px-6 py-3 rounded-full shadow-2xl opacity-0 transition-opacity duration-300 z-[70] pointer-events-none text-sm"></div>

    <script>
        let masterData = [];
        let headers = [];
        let currentSearchResults = [];
        let isCloudConnected = false;

        // --- INIT ---
        window.addEventListener('load', async () => {
            setTimeout(async () => {
                const savedConfig = localStorage.getItem('firebaseConfig');
                let configToUse = savedConfig ? JSON.parse(savedConfig) : undefined;

                if (await window.initFirebaseApp(configToUse)) {
                    isCloudConnected = true;
                    updateStatusUI(true, 'กำลังโหลดข้อมูล...');
                    
                    const result = await window.loadFromFirestore();
                    
                    if (result === null) {
                        updateStatusUI(false, 'สิทธิ์ไม่ผ่าน (โหลดไฟล์เองได้)');
                    } else if (result.data && result.data.length > 0) {
                        masterData = result.data;
                        headers = Object.keys(masterData[0]);
                        
                        updateStatusUI(true, `พร้อมใช้งาน (${masterData.length} รายการ)`);
                        showToast(`☁️ ข้อมูล Cloud พร้อมแล้ว`);
                        
                        document.getElementById('admin-record-count').innerText = masterData.length.toLocaleString();
                        if(result.updatedAt) {
                            const date = new Date(result.updatedAt);
                            document.getElementById('admin-last-update').innerText = date.toLocaleString('th-TH');
                        }
                    } else {
                        updateStatusUI(true, 'ฐานข้อมูลว่างเปล่า');
                        showToast('ℹ️ ฐานข้อมูลว่างเปล่า');
                        document.getElementById('admin-record-count').innerText = "0";
                    }
                }
            }, 1000);
        });

        // --- ADMIN WIDGET LOGIC ---
        let isAdminPanelOpen = false;

        function toggleAdminPanel() {
            const panel = document.getElementById('admin-panel');
            isAdminPanelOpen = !isAdminPanelOpen;
            if (isAdminPanelOpen) {
                panel.classList.remove('admin-panel-enter');
                panel.classList.add('admin-panel-active');
            } else {
                panel.classList.add('admin-panel-enter');
                panel.classList.remove('admin-panel-active');
            }
        }

        function requestAdminAccess() {
            toggleModal('password-modal');
            setTimeout(() => {
                const input = document.getElementById('admin-password-input');
                input.value = ''; input.focus();
            }, 100);
        }

        function checkAdminPassword() {
            const pass = document.getElementById('admin-password-input').value;
            if (pass === 'NYB2') {
                toggleModal('password-modal');
                document.getElementById('admin-locked').classList.add('hidden');
                document.getElementById('admin-unlocked').classList.remove('hidden');
                document.getElementById('admin-logout-btn').classList.remove('hidden');
                showToast('🔓 ยินดีต้อนรับ Admin');
            } else {
                const input = document.getElementById('admin-password-input');
                input.classList.add('border-red-500', 'ring-2', 'ring-red-200');
                setTimeout(() => input.classList.remove('border-red-500', 'ring-2', 'ring-red-200'), 500);
                showToast('❌ รหัสผ่านไม่ถูกต้อง');
            }
        }

        function logoutAdmin() {
            document.getElementById('admin-locked').classList.remove('hidden');
            document.getElementById('admin-unlocked').classList.add('hidden');
            document.getElementById('admin-logout-btn').classList.add('hidden');
            showToast('🔒 ออกจากระบบแล้ว');
        }

        document.getElementById('admin-password-input').addEventListener('keypress', function (e) {
            if (e.key === 'Enter') checkAdminPassword();
        });

        // --- UPLOAD LOGIC ---
        function handleFileSelect(input) {
            if(input.files[0]) {
                document.getElementById('master-file-name').textContent = input.files[0].name;
                const btn = document.getElementById('btn-upload-db');
                btn.disabled = false;
                btn.classList.remove('opacity-50', 'cursor-not-allowed');
            }
        }

        async function triggerDbUpload() {
            if (!isCloudConnected) { alert('ยังไม่เชื่อมต่อ Cloud'); return; }
            const input = document.getElementById('master-db-input');
            const file = input.files[0];
            const reader = new FileReader();
            reader.onload = async (e) => {
                const data = new Uint8Array(e.target.result);
                const workbook = XLSX.read(data, { type: 'array' });
                const json = XLSX.utils.sheet_to_json(workbook.Sheets[workbook.SheetNames[0]], { defval: "" });

                if (json.length === 0) { alert('ไฟล์ไม่มีข้อมูล'); return; }
                if (JSON.stringify(json).length > 950000) { 
                   if(!confirm('ไฟล์ใหญ่เกิน 1MB อาจมีปัญหาโควต้าฟรี ยืนยัน?')) return;
                }

                showToast('<i class="fa-solid fa-spinner fa-spin"></i> กำลังอัปโหลด...');
                const success = await window.saveToFirestore(json);
                
                if (success) {
                    masterData = json;
                    headers = Object.keys(masterData[0]);
                    updateStatusUI(true, `อัปเดตแล้ว (${masterData.length} รายการ)`);
                    
                    document.getElementById('admin-record-count').innerText = masterData.length.toLocaleString();
                    document.getElementById('admin-last-update').innerText = new Date().toLocaleString('th-TH');
                    
                    // Reset Input
                    input.value = '';
                    document.getElementById('master-file-name').textContent = 'เลือกไฟล์ Excel...';
                    document.getElementById('btn-upload-db').disabled = true;
                    document.getElementById('btn-upload-db').classList.add('opacity-50', 'cursor-not-allowed');

                    showToast('✅ อัปโหลดสำเร็จ!');
                } else {
                    showToast('❌ อัปโหลดล้มเหลว');
                }
            };
            reader.readAsArrayBuffer(file);
        }

        async function deleteDatabase() {
            if(!confirm('ยืนยันลบข้อมูลทั้งหมดบน Cloud?')) return;
            showToast('<i class="fa-solid fa-spinner fa-spin"></i> กำลังลบ...');
            const success = await window.deleteFromFirestore();
            if(success) {
                masterData = []; headers = [];
                updateStatusUI(true, 'ฐานข้อมูลว่างเปล่า');
                document.getElementById('admin-record-count').innerText = "0";
                document.getElementById('admin-last-update').innerText = "-";
                showToast('🗑️ ลบข้อมูลเรียบร้อย');
            } else { showToast('❌ ลบไม่สำเร็จ'); }
        }

        // --- SEARCH & LOCAL ---
        function useLocalFile(input) {
            const file = input.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = (e) => {
                const data = new Uint8Array(e.target.result);
                const workbook = XLSX.read(data, { type: 'array' });
                const json = XLSX.utils.sheet_to_json(workbook.Sheets[workbook.SheetNames[0]], { defval: "" });
                
                if (json.length > 0) {
                    masterData = json;
                    headers = Object.keys(masterData[0]);
                    const info = document.getElementById('data-source-info');
                    info.innerHTML = `<i class="fa-solid fa-folder-open text-orange-500"></i> โหมด Local (${file.name})`;
                    info.className = "text-center text-xs text-orange-600 mt-2 font-bold";
                    showToast(`📂 ใช้ไฟล์ Local (${json.length} รายการ)`);
                }
            };
            reader.readAsArrayBuffer(file);
        }

        function performBatchSearch() {
            if (masterData.length === 0) { showToast('⚠️ ไม่พบข้อมูล'); return; }
            const terms = document.getElementById('search-textarea').value.trim().split('\n').map(s => s.trim().toLowerCase()).filter(s => s);
            if (terms.length === 0) return;

            const results = masterData.filter(row => 
                Object.values(row).some(val => terms.some(term => String(val).toLowerCase().includes(term)))
            );
            renderResults(results);
            showToast(`🔍 เจอ ${results.length} รายการ`);
        }

        function renderResults(data) {
            currentSearchResults = data;
            document.getElementById('result-section').classList.remove('hidden');
            document.getElementById('found-count').textContent = data.length;
            const thead = document.getElementById('table-header');
            const tbody = document.getElementById('table-body');
            thead.innerHTML = ''; tbody.innerHTML = '';
            
            if (headers.length === 0 && data.length > 0) headers = Object.keys(data[0]);

            headers.forEach(h => thead.innerHTML += `<th class="px-6 py-3 text-left text-xs font-bold text-gray-500 uppercase bg-gray-100">${h}</th>`);
            data.slice(0, 500).forEach((row, i) => {
                let html = `<tr class="${i%2===0?'bg-white':'bg-gray-50'}">`;
                headers.forEach(h => html += `<td class="px-6 py-4 text-sm text-gray-700 whitespace-nowrap">${row[h]}</td>`);
                tbody.innerHTML += html + '</tr>';
            });
        }

        function exportToExcel() {
            if (!currentSearchResults || currentSearchResults.length === 0) {
                showToast('⚠️ ไม่มีข้อมูลให้ Export');
                return;
            }
            
            const ws = XLSX.utils.json_to_sheet(currentSearchResults);
            const wb = XLSX.utils.book_new();
            XLSX.utils.book_append_sheet(wb, ws, "Search Results");
            
            const date = new Date();
            const timestamp = `${date.getDate()}-${date.getMonth()+1}-${date.getFullYear()}_${date.getHours()}-${date.getMinutes()}`;
            XLSX.writeFile(wb, `Search_Results_${timestamp}.xlsx`);
            
            showToast('✅ Export เรียบร้อย');
        }

        // --- UTILS ---
        function saveSettings() {
            const input = document.getElementById('firebase-config-input').value;
            try {
                const config = (new Function("return " + input))(); 
                if (config && config.apiKey) { localStorage.setItem('firebaseConfig', JSON.stringify(config)); location.reload(); }
            } catch (e) { alert('Config ไม่ถูกต้อง'); }
        }

        function updateStatusUI(online, text) {
            const el = document.getElementById('connection-status');
            const info = document.getElementById('data-source-info');
            if (online) {
                el.innerHTML = `<div class="w-2 h-2 rounded-full bg-green-500"></div> Online`;
                el.classList.replace('text-gray-400', 'text-green-600');
                if(!text.includes('Local')) {
                    info.innerHTML = `<i class="fa-solid fa-cloud text-blue-500"></i> ${text}`;
                    info.className = "text-center text-xs text-blue-600 mt-2 font-medium";
                }
            } else {
                el.innerHTML = `<div class="w-2 h-2 rounded-full bg-red-500"></div> Error`;
                info.innerHTML = `<i class="fa-solid fa-triangle-exclamation text-red-500"></i> ${text}`;
                info.className = "text-center text-xs text-red-500 mt-2 font-bold cursor-pointer";
                info.onclick = () => toggleModal('rules-help-modal');
            }
        }

        function copyRules() {
            const code = document.getElementById('rules-code').innerText;
            const textArea = document.createElement("textarea");
            textArea.value = code;
            textArea.style.position = "fixed"; textArea.style.left = "-9999px";
            document.body.appendChild(textArea);
            textArea.select();
            try { document.execCommand('copy'); alert('คัดลอกโค้ดแล้ว!'); } catch (err) { prompt("กด Ctrl+C:", code); }
            document.body.removeChild(textArea);
        }

        function toggleModal(id) {
            const m = document.getElementById(id);
            m.classList.toggle('opacity-0'); m.classList.toggle('pointer-events-none');
            if (!m.classList.contains('opacity-0') && id === 'settings-modal') {
                const saved = localStorage.getItem('firebaseConfig');
                document.getElementById('firebase-config-input').value = saved || JSON.stringify({ info: "Using Default Config" }, null, 2);
            }
        }
        function showToast(msg) {
            const t = document.getElementById('toast');
            t.innerHTML = msg; t.classList.remove('opacity-0');
            setTimeout(() => t.classList.add('opacity-0'), 3000);
        }
    </script>
</body>
</html>
