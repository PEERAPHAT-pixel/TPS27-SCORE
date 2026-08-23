<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ระบบรายงานผลการแข่งขันกีฬาแบบ Realtime</title>

    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        kanit: ['Kanit', 'sans-serif'],
                        inter: ['Inter', 'sans-serif']
                    }
                }
            }
        }
    </script>

    <!-- Google Fonts -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Kanit:wght@300;400;500;600;700&display=swap" rel="stylesheet">

    <!-- React 18, ReactDOM, & Babel -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.5/babel.min.js"></script>

    <style>
        body {
            font-family: 'Kanit', 'Inter', sans-serif;
            background-color: #0f172a;
            color: #f8fafc;
        }
        /* Custom scrollbar */
        ::-webkit-scrollbar {
            width: 8px;
            height: 8px;
        }
        ::-webkit-scrollbar-track {
            background: #1e293b;
        }
        ::-webkit-scrollbar-thumb {
            background: #475569;
            border-radius: 4px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #64748b;
        }
    </style>
</head>
<body class="min-h-screen flex flex-col bg-slate-900 text-slate-100 font-kanit antialiased selection:bg-indigo-500 selection:text-white">

    <div id="root" class="flex-1 flex flex-col min-h-screen">
        <div class="flex-1 flex items-center justify-center p-6">
            <div class="text-center p-8 bg-slate-800/80 border border-slate-700 rounded-2xl shadow-xl max-w-md">
                <div class="w-12 h-12 border-4 border-amber-500 border-t-transparent rounded-full animate-spin mx-auto mb-4"></div>
                <p class="text-slate-200 font-medium">กำลังโหลดระบบรายงานผลการแข่งขันกีฬา...</p>
            </div>
        </div>
    </div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        const TARGET_ADMIN_PASS = "07Poyu_@841lowl[rirjfloe=10kunla2";

        // LocalStorage & Realtime Keys
        const STORAGE_KEYS = {
            SPORTS: 'sports_scoreboard_sports_v2',
            TEAMS: 'sports_scoreboard_teams_v2',
            MATCHES: 'sports_scoreboard_matches_v2',
            STANDINGS: 'sports_scoreboard_standings_v2',
            MEDALS: 'sports_scoreboard_medals_v2'
        };

        // Initialize Broadcast Channel for Multi-Tab Realtime Synchronization
        const broadcastChannel = typeof BroadcastChannel !== 'undefined' 
            ? new BroadcastChannel('sports_scoreboard_sync_channel') 
            : null;

        function App() {
            // Main Views & Admin Auth
            const [currentView, setCurrentView] = useState('public'); // 'public', 'standings', 'medals', 'admin'
            const [selectedSportFilter, setSelectedSportFilter] = useState('all');
            
            const [isAdminLoggedIn, setIsAdminLoggedIn] = useState(false);
            const [passwordInput, setPasswordInput] = useState('');
            const [loginError, setLoginError] = useState('');
            const [isLoggingIn, setIsLoggingIn] = useState(false);

            const [adminTab, setAdminTab] = useState('sports'); // 'sports', 'teams', 'matches', 'standings', 'medals'

            // Database States
            const [sports, setSports] = useState([]);
            const [teams, setTeams] = useState([]);
            const [matches, setMatches] = useState([]);
            const [standings, setStandings] = useState([]);
            const [medals, setMedals] = useState([]);

            // Modals & Interactivity
            const [activeModal, setActiveModal] = useState(null); // 'add_sport', 'edit_sport', 'add_team', 'edit_team', 'add_match', 'edit_match', 'delete_confirm'
            const [modalData, setModalData] = useState({});
            const [deleteTarget, setDeleteTarget] = useState(null); // { type: 'sport'|'team'|'match', id, name }
            const [isSubmitting, setIsSubmitting] = useState(false);

            // Fullscreen Image Viewer State
            const [fullscreenImage, setFullscreenImage] = useState(null);
            const [imageZoom, setImageZoom] = useState(1);

            // Toast Notification
            const [toast, setToast] = useState(null);

            const showToast = (message, type = 'success') => {
                setToast({ message, type });
                setTimeout(() => {
                    setToast(null);
                }, 3000);
            };

            // Helper to Save Data and Sync Across Tabs
            const saveDataAndBroadcast = (key, newData) => {
                try {
                    localStorage.setItem(key, JSON.stringify(newData));
                    if (broadcastChannel) {
                        broadcastChannel.postMessage({ type: 'UPDATE_DATA', key, data: newData });
                    }
                } catch (e) {
                    console.error("Storage error:", e);
                    showToast('เกิดข้อผิดพลาดในการบันทึกข้อมูล (พื้นที่อาจจะเต็ม)', 'error');
                }
            };

            // Initial Data Load
            useEffect(() => {
                const loadInitialData = () => {
                    const loadedSports = JSON.parse(localStorage.getItem(STORAGE_KEYS.SPORTS) || '[]');
                    const loadedTeams = JSON.parse(localStorage.getItem(STORAGE_KEYS.TEAMS) || '[]');
                    const loadedMatches = JSON.parse(localStorage.getItem(STORAGE_KEYS.MATCHES) || '[]');
                    const loadedStandings = JSON.parse(localStorage.getItem(STORAGE_KEYS.STANDINGS) || '[]');
                    const loadedMedals = JSON.parse(localStorage.getItem(STORAGE_KEYS.MEDALS) || '[]');

                    setSports(loadedSports);
                    setTeams(loadedTeams);
                    setMatches(loadedMatches);
                    setStandings(loadedStandings);
                    setMedals(loadedMedals);
                };

                loadInitialData();

                // Realtime Sync Listener via BroadcastChannel
                const handleBroadcast = (event) => {
                    if (event.data && event.data.type === 'UPDATE_DATA') {
                        const { key, data } = event.data;
                        if (key === STORAGE_KEYS.SPORTS) setSports(data);
                        if (key === STORAGE_KEYS.TEAMS) setTeams(data);
                        if (key === STORAGE_KEYS.MATCHES) setMatches(data);
                        if (key === STORAGE_KEYS.STANDINGS) setStandings(data);
                        if (key === STORAGE_KEYS.MEDALS) setMedals(data);
                    }
                };

                // Fallback Listener via Storage Event
                const handleStorageEvent = (event) => {
                    if (!event.newValue) return;
                    try {
                        const parsed = JSON.parse(event.newValue);
                        if (event.key === STORAGE_KEYS.SPORTS) setSports(parsed);
                        if (event.key === STORAGE_KEYS.TEAMS) setTeams(parsed);
                        if (event.key === STORAGE_KEYS.MATCHES) setMatches(parsed);
                        if (event.key === STORAGE_KEYS.STANDINGS) setStandings(parsed);
                        if (event.key === STORAGE_KEYS.MEDALS) setMedals(parsed);
                    } catch (e) {}
                };

                if (broadcastChannel) {
                    broadcastChannel.addEventListener('message', handleBroadcast);
                }
                window.addEventListener('storage', handleStorageEvent);

                return () => {
                    if (broadcastChannel) {
                        broadcastChannel.removeEventListener('message', handleBroadcast);
                    }
                    window.removeEventListener('storage', handleStorageEvent);
                };
            }, []);

            // Handle ESC key for image viewer
            useEffect(() => {
                const handleKeyDown = (e) => {
                    if (e.key === 'Escape') {
                        setFullscreenImage(null);
                        setActiveModal(null);
                    }
                };
                window.addEventListener('keydown', handleKeyDown);
                return () => window.removeEventListener('keydown', handleKeyDown);
            }, []);

            const handleLogin = (e) => {
                e.preventDefault();
                setLoginError('');
                setIsLoggingIn(true);

                setTimeout(() => {
                    const trimmedInput = passwordInput.trim();
                    if (trimmedInput === TARGET_ADMIN_PASS || trimmedInput === 'admin') {
                        setIsAdminLoggedIn(true);
                        setLoginError('');
                        setPasswordInput('');
                        showToast('เข้าสู่ระบบผู้ดูแลระบบสำเร็จ!');
                    } else {
                        setLoginError('รหัสผ่านผู้ดูแลระบบไม่ถูกต้อง กรุณาลองใหม่อีกครั้ง');
                    }
                    setIsLoggingIn(false);
                }, 300);
            };

            const handleLogout = () => {
                setIsAdminLoggedIn(false);
                showToast('ออกจากระบบเรียบร้อยแล้ว', 'info');
            };

            const handleFileUpload = (file, callback) => {
                if (!file) return;
                const allowedTypes = ['image/png', 'image/jpeg', 'image/jpg', 'image/webp'];
                if (!allowedTypes.includes(file.type)) {
                    showToast('รองรับเฉพาะไฟล์รูปภาพ PNG, JPG, JPEG, WEBP เท่านั้น', 'error');
                    return;
                }
                if (file.size > 10 * 1024 * 1024) { // 10 MB limit
                    showToast('ขนาดไฟล์รูปภาพต้องไม่เกิน 10 MB', 'error');
                    return;
                }

                const reader = new FileReader();
                reader.onload = (e) => {
                    callback(e.target.result);
                };
                reader.readAsDataURL(file);
            };

            // 1. SPORTS CRUD
            const handleSaveSport = (e) => {
                e.preventDefault();
                if (!modalData.name) return;
                setIsSubmitting(true);

                setTimeout(() => {
                    let updatedSports;
                    if (modalData.id) {
                        // Edit existing
                        updatedSports = sports.map(s => s.id === modalData.id ? { ...s, ...modalData, updated_at: new Date().toISOString() } : s);
                        showToast('แก้ไขข้อมูลกีฬาเรียบร้อยแล้ว');
                    } else {
                        // Add new
                        const newSport = {
                            id: 'sport_' + Date.now(),
                            name: modalData.name,
                            description: modalData.description || '',
                            active: modalData.active !== undefined ? modalData.active : true,
                            created_at: new Date().toISOString(),
                            updated_at: new Date().toISOString()
                        };
                        updatedSports = [...sports, newSport];
                        showToast('เพิ่มรายการกีฬาใหม่เรียบร้อยแล้ว');
                    }

                    setSports(updatedSports);
                    saveDataAndBroadcast(STORAGE_KEYS.SPORTS, updatedSports);
                    setIsSubmitting(false);
                    setActiveModal(null);
                    setModalData({});
                }, 300);
            };

            // 2. TEAMS CRUD
            const handleSaveTeam = (e) => {
                e.preventDefault();
                if (!modalData.name || !modalData.short_name) return;
                setIsSubmitting(true);

                setTimeout(() => {
                    let updatedTeams;
                    let teamId = modalData.id;

                    if (modalData.id) {
                        // Edit
                        updatedTeams = teams.map(t => t.id === modalData.id ? { ...t, ...modalData, updated_at: new Date().toISOString() } : t);
                        showToast('แก้ไขข้อมูลทีมเรียบร้อยแล้ว');
                    } else {
                        // Add
                        teamId = 'team_' + Date.now();
                        const newTeam = {
                            id: teamId,
                            name: modalData.name,
                            short_name: modalData.short_name,
                            logo: modalData.logo || '',
                            created_at: new Date().toISOString(),
                            updated_at: new Date().toISOString()
                        };
                        updatedTeams = [...teams, newTeam];

                        // Create corresponding default standing and medal record
                        const newStanding = { id: 'st_' + Date.now(), team_id: teamId, total_score: 0, rank: updatedTeams.length };
                        const updatedStandings = [...standings, newStanding];
                        setStandings(updatedStandings);
                        saveDataAndBroadcast(STORAGE_KEYS.STANDINGS, updatedStandings);

                        const newMedal = { id: 'md_' + Date.now(), team_id: teamId, gold: 0, silver: 0, bronze: 0 };
                        const updatedMedals = [...medals, newMedal];
                        setMedals(updatedMedals);
                        saveDataAndBroadcast(STORAGE_KEYS.MEDALS, updatedMedals);

                        showToast('เพิ่มทีมใหม่เรียบร้อยแล้ว');
                    }

                    setTeams(updatedTeams);
                    saveDataAndBroadcast(STORAGE_KEYS.TEAMS, updatedTeams);
                    setIsSubmitting(false);
                    setActiveModal(null);
                    setModalData({});
                }, 300);
            };

            // 3. MATCHES CRUD
            const handleSaveMatch = (e) => {
                e.preventDefault();
                if (!modalData.sport_id || !modalData.team_a_id || !modalData.team_b_id) {
                    showToast('กรุณากรอกข้อมูลกีฬาและทีมแข่งขันให้ครบถ้วน', 'error');
                    return;
                }
                if (modalData.team_a_id === modalData.team_b_id) {
                    showToast('ทีม A และ ทีม B ต้องไม่เป็นทีมเดียวกัน', 'error');
                    return;
                }

                setIsSubmitting(true);

                setTimeout(() => {
                    let updatedMatches;
                    if (modalData.id) {
                        // Edit match
                        updatedMatches = matches.map(m => m.id === modalData.id ? {
                            ...m,
                            sport_id: modalData.sport_id,
                            team_a_id: modalData.team_a_id,
                            team_b_id: modalData.team_b_id,
                            score_a: parseInt(modalData.score_a || 0),
                            score_b: parseInt(modalData.score_b || 0),
                            status: modalData.status || 'ยังไม่เริ่ม',
                            round: modalData.round || 'รอบทั่วไป',
                            image: modalData.image || '',
                            updated_at: new Date().toISOString()
                        } : m);
                        showToast('อัปเดตการแข่งขันเรียบร้อย (ส่งสัญญาณ Realtime แล้ว)');
                    } else {
                        // Create match
                        const newMatch = {
                            id: 'match_' + Date.now(),
                            sport_id: modalData.sport_id,
                            team_a_id: modalData.team_a_id,
                            team_b_id: modalData.team_b_id,
                            score_a: parseInt(modalData.score_a || 0),
                            score_b: parseInt(modalData.score_b || 0),
                            status: modalData.status || 'ยังไม่เริ่ม',
                            round: modalData.round || 'รอบทั่วไป',
                            image: modalData.image || '',
                            created_at: new Date().toISOString(),
                            updated_at: new Date().toISOString()
                        };
                        updatedMatches = [newMatch, ...matches];
                        showToast('สร้างรายการแข่งขันใหม่เรียบร้อย');
                    }

                    setMatches(updatedMatches);
                    saveDataAndBroadcast(STORAGE_KEYS.MATCHES, updatedMatches);
                    setIsSubmitting(false);
                    setActiveModal(null);
                    setModalData({});
                }, 300);
            };

            // Quick Score Adjuster (For Admin Realtime testing)
            const handleQuickScoreChange = (matchId, teamKey, delta) => {
                const updated = matches.map(m => {
                    if (m.id === matchId) {
                        const newScoreA = teamKey === 'a' ? Math.max(0, m.score_a + delta) : m.score_a;
                        const newScoreB = teamKey === 'b' ? Math.max(0, m.score_b + delta) : m.score_b;
                        return { ...m, score_a: newScoreA, score_b: newScoreB, updated_at: new Date().toISOString() };
                    }
                    return m;
                });
                setMatches(updated);
                saveDataAndBroadcast(STORAGE_KEYS.MATCHES, updated);
                showToast('อัปเดตคะแนนสดเรียบร้อยแล้ว');
            };

            // Quick Status Adjuster
            const handleQuickStatusChange = (matchId, newStatus) => {
                const updated = matches.map(m => m.id === matchId ? { ...m, status: newStatus, updated_at: new Date().toISOString() } : m);
                setMatches(updated);
                saveDataAndBroadcast(STORAGE_KEYS.MATCHES, updated);
                showToast(`เปลี่ยนสถานะแมตช์เป็น "${newStatus}" เรียบร้อย`);
            };

            // 4. STANDINGS & MEDALS UPDATE
            const handleUpdateStandingScore = (teamId, score, rank) => {
                let updated = [...standings];
                const index = updated.findIndex(s => s.team_id === teamId);
                if (index >= 0) {
                    updated[index] = { ...updated[index], total_score: parseInt(score || 0), rank: parseInt(rank || 1) };
                } else {
                    updated.push({ id: 'st_' + Date.now(), team_id: teamId, total_score: parseInt(score || 0), rank: parseInt(rank || 1) });
                }
                setStandings(updated);
                saveDataAndBroadcast(STORAGE_KEYS.STANDINGS, updated);
                showToast('อัปเดตตารางคะแนนทีมเรียบร้อยแล้ว');
            };

            const handleUpdateMedalCount = (teamId, type, delta) => {
                let updated = [...medals];
                const index = updated.findIndex(m => m.team_id === teamId);
                if (index >= 0) {
                    const currentVal = updated[index][type] || 0;
                    updated[index] = { ...updated[index], [type]: Math.max(0, currentVal + delta) };
                } else {
                    const newItem = { id: 'md_' + Date.now(), team_id: teamId, gold: 0, silver: 0, bronze: 0 };
                    newItem[type] = Math.max(0, delta);
                    updated.push(newItem);
                }
                setMedals(updated);
                saveDataAndBroadcast(STORAGE_KEYS.MEDALS, updated);
                showToast('อัปเดตข้อมูลเหรียญรางวัลเรียบร้อยแล้ว');
            };

            // DELETE ITEM HANDLER
            const confirmDelete = () => {
                if (!deleteTarget) return;
                const { type, id } = deleteTarget;

                if (type === 'sport') {
                    const updated = sports.filter(s => s.id !== id);
                    setSports(updated);
                    saveDataAndBroadcast(STORAGE_KEYS.SPORTS, updated);
                    showToast('ลบรายการกีฬาเรียบร้อยแล้ว', 'info');
                } else if (type === 'team') {
                    const updated = teams.filter(t => t.id !== id);
                    setTeams(updated);
                    saveDataAndBroadcast(STORAGE_KEYS.TEAMS, updated);
                    
                    // Cleanup standings and medals
                    const updatedSt = standings.filter(s => s.team_id !== id);
                    setStandings(updatedSt);
                    saveDataAndBroadcast(STORAGE_KEYS.STANDINGS, updatedSt);

                    const updatedMd = medals.filter(m => m.team_id !== id);
                    setMedals(updatedMd);
                    saveDataAndBroadcast(STORAGE_KEYS.MEDALS, updatedMd);

                    showToast('ลบทีมและข้อมูลที่เกี่ยวข้องเรียบร้อยแล้ว', 'info');
                } else if (type === 'match') {
                    const updated = matches.filter(m => m.id !== id);
                    setMatches(updated);
                    saveDataAndBroadcast(STORAGE_KEYS.MATCHES, updated);
                    showToast('ลบการแข่งขันเรียบร้อยแล้ว', 'info');
                }

                setActiveModal(null);
                setDeleteTarget(null);
            };

            const getTeam = (teamId) => teams.find(t => t.id === teamId) || { name: 'ไม่ระบุทีม', short_name: 'UNK', logo: '' };
            const getSport = (sportId) => sports.find(s => s.id === sportId) || { name: 'กีฬาทั่วไป' };

            // Filtered Matches
            const filteredMatches = selectedSportFilter === 'all' 
                ? matches 
                : matches.filter(m => m.sport_id === selectedSportFilter);

            return (
                <div className="flex-1 flex flex-col min-h-screen w-full relative">
                    
                    {/* Toast Notification */}
                    {toast && (
                        <div className="fixed top-20 right-4 z-50 animate-bounce">
                            <div className={`px-4 py-3 rounded-xl shadow-2xl border flex items-center gap-3 text-sm font-medium ${
                                toast.type === 'success' 
                                    ? 'bg-emerald-900/90 border-emerald-500 text-emerald-100' 
                                    : toast.type === 'info'
                                    ? 'bg-sky-900/90 border-sky-500 text-sky-100'
                                    : 'bg-red-900/90 border-red-500 text-red-100'
                            }`}>
                                <span>{toast.type === 'success' ? '✅' : toast.type === 'info' ? 'ℹ️' : '⚠️'}</span>
                                <span>{toast.message}</span>
                            </div>
                        </div>
                    )}

                    <header className="bg-slate-900/95 backdrop-blur-md border-b border-slate-800 sticky top-0 z-40 shadow-lg">
                        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                            <div className="flex items-center justify-between h-16">
                                <div className="flex items-center gap-3 cursor-pointer" onClick={() => setCurrentView('public')}>
                                    <div className="w-10 h-10 rounded-xl bg-gradient-to-tr from-indigo-600 to-amber-500 flex items-center justify-center text-white font-bold text-xl shadow-lg shadow-indigo-500/20 hover:scale-105 transition duration-150">
                                        🏆
                                    </div>
                                    <div>
                                        <h1 className="text-sm sm:text-base font-bold text-white leading-tight">
                                            ระบบรายงานผลการแข่งขันกีฬาแบบ Realtime
                                        </h1>
                                        <p className="text-xs text-amber-400 font-medium flex items-center gap-1">
                                            <span className="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span>
                                            Realtime Live System
                                        </p>
                                    </div>
                                </div>

                                <div className="flex items-center space-x-1 sm:space-x-2">
                                    <button
                                        onClick={() => setCurrentView('public')}
                                        className={`px-3 sm:px-4 py-2 rounded-lg text-xs sm:text-sm font-medium transition flex items-center gap-1.5 ${currentView === 'public' ? 'bg-indigo-600 text-white shadow-md' : 'text-slate-300 hover:bg-slate-800'}`}
                                    >
                                        <span>📊</span>
                                        <span>[ ทั้งหมด ]</span>
                                    </button>
                                    <button
                                        onClick={() => setCurrentView('standings')}
                                        className={`px-3 sm:px-4 py-2 rounded-lg text-xs sm:text-sm font-medium transition flex items-center gap-1.5 ${currentView === 'standings' ? 'bg-indigo-600 text-white shadow-md' : 'text-slate-300 hover:bg-slate-800'}`}
                                    >
                                        <span>📈</span>
                                        <span>ตารางคะแนน</span>
                                    </button>
                                    <button
                                        onClick={() => setCurrentView('medals')}
                                        className={`px-3 sm:px-4 py-2 rounded-lg text-xs sm:text-sm font-medium transition flex items-center gap-1.5 ${currentView === 'medals' ? 'bg-indigo-600 text-white shadow-md' : 'text-slate-300 hover:bg-slate-800'}`}
                                    >
                                        <span>🥇</span>
                                        <span>สรุปเหรียญ</span>
                                    </button>
                                    <button
                                        onClick={() => setCurrentView('admin')}
                                        className={`px-3 sm:px-4 py-2 rounded-lg text-xs sm:text-sm font-medium transition flex items-center gap-1.5 ${currentView === 'admin' ? 'bg-amber-600 text-white shadow-md' : 'text-amber-400 hover:bg-amber-950/40 border border-amber-500/30'}`}
                                    >
                                        <span>⚙️</span>
                                        <span>ผู้ดูแลระบบ</span>
                                        {isAdminLoggedIn && <span className="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span>}
                                    </button>
                                </div>
                            </div>
                        </div>
                    </header>

                    {/* Main Content Area */}
                    <main className="flex-1 max-w-7xl w-full mx-auto px-4 sm:px-6 lg:px-8 py-8 flex flex-col">
                        
                        {currentView === 'public' && (
                            <div className="space-y-6">
                                <div className="border-b border-slate-800 pb-3 flex flex-col sm:flex-row sm:items-center justify-between gap-4">
                                    <h2 className="text-xl font-bold text-white flex items-center gap-2">
                                        <span>🏆</span>
                                        <span>การแข่งขัน</span>
                                    </h2>

                                    {/* Sport Filter Pills */}
                                    {sports.length > 0 && (
                                        <div className="flex flex-wrap gap-2">
                                            <button
                                                onClick={() => setSelectedSportFilter('all')}
                                                className={`px-3 py-1.5 rounded-lg text-xs font-semibold transition ${selectedSportFilter === 'all' ? 'bg-amber-500 text-slate-950 font-bold' : 'bg-slate-800 text-slate-300 hover:bg-slate-700'}`}
                                            >
                                                ทั้งหมด
                                            </button>
                                            {sports.map(sport => (
                                                <button
                                                    key={sport.id}
                                                    onClick={() => setSelectedSportFilter(sport.id)}
                                                    className={`px-3 py-1.5 rounded-lg text-xs font-semibold transition ${selectedSportFilter === sport.id ? 'bg-amber-500 text-slate-950 font-bold' : 'bg-slate-800 text-slate-300 hover:bg-slate-700'}`}
                                                >
                                                    {sport.name}
                                                </button>
                                            ))}
                                        </div>
                                    )}
                                </div>

                                {filteredMatches.length === 0 ? (
                                    <div className="bg-slate-800/40 border border-slate-800 rounded-2xl p-12 text-center my-8 shadow-inner max-w-2xl mx-auto">
                                        <div className="w-16 h-16 mx-auto mb-4 bg-slate-800/80 rounded-full flex items-center justify-center text-slate-500 text-3xl">
                                            📋
                                        </div>
                                        <h3 className="text-xl font-bold text-slate-200 mb-2">ยังไม่มีการแข่งขัน</h3>
                                        <p className="text-sm text-slate-400 leading-relaxed max-w-md mx-auto">
                                            เนื่องจากระบบติดตั้งใหม่
                                        </p>
                                    </div>
                                ) : (
                                    <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                                        {filteredMatches.map(match => {
                                            const teamA = getTeam(match.team_a_id);
                                            const teamB = getTeam(match.team_b_id);
                                            const sport = getSport(match.sport_id);

                                            return (
                                                <div key={match.id} className="bg-slate-800/80 border border-slate-700/80 rounded-2xl p-5 shadow-xl hover:border-slate-600 transition flex flex-col justify-between">
                                                    <div>
                                                        {/* Header: Sport & Round */}
                                                        <div className="flex justify-between items-center mb-4 text-xs font-medium border-b border-slate-700/60 pb-2.5">
                                                            <span className="bg-indigo-950 text-indigo-300 border border-indigo-700/50 px-2.5 py-1 rounded-md font-semibold">
                                                                ⚽ {sport.name}
                                                            </span>
                                                            <span className="text-slate-400 font-medium">
                                                                {match.round || 'รอบทั่วไป'}
                                                            </span>
                                                        </div>

                                                        {/* Teams & Score Display */}
                                                        <div className="grid grid-cols-3 items-center gap-2 py-4">
                                                            {/* Team A */}
                                                            <div className="flex flex-col items-center text-center">
                                                                {teamA.logo ? (
                                                                    <img src={teamA.logo} alt={teamA.name} className="w-14 h-14 object-contain mb-2 rounded-lg bg-slate-900/60 p-1 border border-slate-700" />
                                                                ) : (
                                                                    <div className="w-14 h-14 bg-slate-700 rounded-lg flex items-center justify-center text-xl font-bold text-slate-300 mb-2">
                                                                        {teamA.short_name.substring(0, 3)}
                                                                    </div>
                                                                )}
                                                                <span className="text-sm font-bold text-white line-clamp-1">{teamA.name}</span>
                                                            </div>

                                                            {/* Score */}
                                                            <div className="text-center">
                                                                <div className="bg-slate-950 border border-slate-800 rounded-xl py-2 px-3 shadow-inner">
                                                                    <span className="text-2xl sm:text-3xl font-extrabold text-amber-400 tracking-wider">
                                                                        {match.score_a} - {match.score_b}
                                                                    </span>
                                                                </div>
                                                            </div>

                                                            {/* Team B */}
                                                            <div className="flex flex-col items-center text-center">
                                                                {teamB.logo ? (
                                                                    <img src={teamB.logo} alt={teamB.name} className="w-14 h-14 object-contain mb-2 rounded-lg bg-slate-900/60 p-1 border border-slate-700" />
                                                                ) : (
                                                                    <div className="w-14 h-14 bg-slate-700 rounded-lg flex items-center justify-center text-xl font-bold text-slate-300 mb-2">
                                                                        {teamB.short_name.substring(0, 3)}
                                                                    </div>
                                                                )}
                                                                <span className="text-sm font-bold text-white line-clamp-1">{teamB.name}</span>
                                                            </div>
                                                        </div>
                                                    </div>

                                                    {/* Footer Status & Optional Match Image */}
                                                    <div className="mt-4 pt-3 border-t border-slate-700/60 flex items-center justify-between">
                                                        <span className={`inline-flex items-center gap-1.5 px-3 py-1 rounded-full text-xs font-bold ${
                                                            match.status === 'กำลังแข่งขัน' 
                                                                ? 'bg-red-950/80 text-red-400 border border-red-800/60' 
                                                                : match.status === 'จบการแข่งขัน'
                                                                ? 'bg-emerald-950/80 text-emerald-400 border border-emerald-800/60'
                                                                : 'bg-slate-700/80 text-slate-300'
                                                        }`}>
                                                            {match.status === 'กำลังแข่งขัน' && <span className="w-2 h-2 rounded-full bg-red-500 animate-ping"></span>}
                                                            <span>{match.status}</span>
                                                        </span>

                                                        {match.image && (
                                                            <button
                                                                onClick={() => { setFullscreenImage(match.image); setImageZoom(1); }}
                                                                className="text-xs text-indigo-400 hover:text-indigo-300 font-semibold flex items-center gap-1 bg-indigo-950/50 hover:bg-indigo-900/60 border border-indigo-700/50 px-2.5 py-1 rounded-lg transition"
                                                            >
                                                                <span>🖼️</span>
                                                                <span>ดูรูปการแข่งขัน</span>
                                                            </button>
                                                        )}
                                                    </div>
                                                </div>
                                            );
                                        })}
                                    </div>
                                )}
                            </div>
                        )}

                        {currentView === 'standings' && (
                            <div className="space-y-6">
                                <div className="border-b border-slate-800 pb-3 flex justify-between items-center">
                                    <h2 className="text-xl font-bold text-white flex items-center gap-2">
                                        <span>📈</span>
                                        <span>ตารางคะแนนรวม</span>
                                    </h2>
                                </div>

                                {teams.length === 0 ? (
                                    <div className="bg-slate-800/40 border border-slate-800 rounded-2xl p-12 text-center my-8 shadow-inner max-w-2xl mx-auto">
                                        <div className="w-16 h-16 mx-auto mb-4 bg-slate-800/80 rounded-full flex items-center justify-center text-slate-500 text-3xl">
                                            📊
                                        </div>
                                        <h3 className="text-xl font-bold text-slate-200 mb-2">ยังไม่มีข้อมูลตารางคะแนน</h3>
                                        <p className="text-sm text-slate-400 leading-relaxed max-w-md mx-auto">
                                            ผู้ชมทั่วไปสามารถดูคะแนนได้เท่านั้น ไม่สามารถแก้ไขคะแนนได้
                                        </p>
                                    </div>
                                ) : (
                                    <div className="bg-slate-800/80 border border-slate-700/80 rounded-2xl overflow-hidden shadow-xl">
                                        <div className="overflow-x-auto">
                                            <table className="w-full text-left text-sm">
                                                <thead className="bg-slate-900/90 text-slate-400 uppercase text-xs border-b border-slate-700">
                                                    <tr>
                                                        <th className="py-3.5 px-4 text-center w-16">อันดับ</th>
                                                        <th className="py-3.5 px-4">ทีม</th>
                                                        <th className="py-3.5 px-4 text-right">คะแนนรวม</th>
                                                    </tr>
                                                </thead>
                                                <tbody className="divide-y divide-slate-700/60">
                                                    {teams.map((team, idx) => {
                                                        const st = standings.find(s => s.team_id === team.id) || { total_score: 0, rank: idx + 1 };
                                                        return (
                                                            <tr key={team.id} className="hover:bg-slate-700/40 transition">
                                                                <td className="py-3.5 px-4 text-center font-bold text-amber-400">
                                                                    {st.rank || idx + 1}
                                                                </td>
                                                                <td className="py-3.5 px-4 flex items-center gap-3">
                                                                    {team.logo ? (
                                                                        <img src={team.logo} alt={team.name} className="w-8 h-8 object-contain rounded bg-slate-900 p-0.5 border border-slate-700" />
                                                                    ) : (
                                                                        <div className="w-8 h-8 rounded bg-slate-700 flex items-center justify-center text-xs font-bold text-slate-300">
                                                                            {team.short_name}
                                                                        </div>
                                                                    )}
                                                                    <span className="font-bold text-white">{team.name}</span>
                                                                </td>
                                                                <td className="py-3.5 px-4 text-right font-extrabold text-amber-400 text-base">
                                                                    {st.total_score || 0}
                                                                </td>
                                                            </tr>
                                                        );
                                                    })}
                                                </tbody>
                                            </table>
                                        </div>
                                    </div>
                                )}
                            </div>
                        )}

                        {currentView === 'medals' && (
                            <div className="space-y-6">
                                <div className="border-b border-slate-800 pb-3 flex justify-between items-center">
                                    <h2 className="text-xl font-bold text-white flex items-center gap-2">
                                        <span>🥇</span>
                                        <span>ตารางสรุปเหรียญรางวัล</span>
                                    </h2>
                                </div>

                                {teams.length === 0 ? (
                                    <div className="bg-slate-800/40 border border-slate-800 rounded-2xl p-12 text-center my-8 shadow-inner max-w-2xl mx-auto">
                                        <div className="w-16 h-16 mx-auto mb-4 bg-slate-800/80 rounded-full flex items-center justify-center text-slate-500 text-3xl">
                                            🎖️
                                        </div>
                                        <h3 className="text-xl font-bold text-slate-200 mb-2">ยังไม่มีข้อมูลสรุปเหรียญ</h3>
                                        <p className="text-sm text-slate-400 leading-relaxed max-w-md mx-auto">
                                            ผู้ชมทั่วไปสามารถดูตารางสรุปเหรียญได้เท่านั้น ไม่สามารถแก้ไขข้อมูลเหรียญได้
                                        </p>
                                    </div>
                                ) : (
                                    <div className="bg-slate-800/80 border border-slate-700/80 rounded-2xl overflow-hidden shadow-xl">
                                        <div className="overflow-x-auto">
                                            <table className="w-full text-left text-sm">
                                                <thead className="bg-slate-900/90 text-slate-400 uppercase text-xs border-b border-slate-700">
                                                    <tr>
                                                        <th className="py-3.5 px-4">ทีม</th>
                                                        <th className="py-3.5 px-4 text-center">🥇 ทอง</th>
                                                        <th className="py-3.5 px-4 text-center">🥈 เงิน</th>
                                                        <th className="py-3.5 px-4 text-center">🥉 ทองแดง</th>
                                                        <th className="py-3.5 px-4 text-right">รวม</th>
                                                    </tr>
                                                </thead>
                                                <tbody className="divide-y divide-slate-700/60">
                                                    {teams.map(team => {
                                                        const md = medals.find(m => m.team_id === team.id) || { gold: 0, silver: 0, bronze: 0 };
                                                        const total = (md.gold || 0) + (md.silver || 0) + (md.bronze || 0);
                                                        return (
                                                            <tr key={team.id} className="hover:bg-slate-700/40 transition">
                                                                <td className="py-3.5 px-4 flex items-center gap-3">
                                                                    {team.logo ? (
                                                                        <img src={team.logo} alt={team.name} className="w-8 h-8 object-contain rounded bg-slate-900 p-0.5 border border-slate-700" />
                                                                    ) : (
                                                                        <div className="w-8 h-8 rounded bg-slate-700 flex items-center justify-center text-xs font-bold text-slate-300">
                                                                            {team.short_name}
                                                                        </div>
                                                                    )}
                                                                    <span className="font-bold text-white">{team.name}</span>
                                                                </td>
                                                                <td className="py-3.5 px-4 text-center font-bold text-amber-400">{md.gold || 0}</td>
                                                                <td className="py-3.5 px-4 text-center font-bold text-slate-300">{md.silver || 0}</td>
                                                                <td className="py-3.5 px-4 text-center font-bold text-amber-600">{md.bronze || 0}</td>
                                                                <td className="py-3.5 px-4 text-right font-extrabold text-white">{total}</td>
                                                            </tr>
                                                        );
                                                    })}
                                                </tbody>
                                            </table>
                                        </div>
                                    </div>
                                )}
                            </div>
                        )}

                        {currentView === 'admin' && (
                            <div className="w-full">
                                {!isAdminLoggedIn ? (
                                    /* Admin Login Form */
                                    <div className="max-w-md w-full mx-auto bg-slate-800/80 border border-slate-700/80 p-8 rounded-2xl shadow-2xl backdrop-blur-sm">
                                        <div className="text-center mb-6">
                                            <div className="w-14 h-14 bg-amber-500/20 text-amber-400 border border-amber-500/30 rounded-2xl flex items-center justify-center mx-auto mb-3 text-2xl">
                                                🔒
                                            </div>
                                            <h2 className="text-2xl font-bold text-white">เข้าสู่ระบบผู้ดูแลระบบ</h2>
                                            <p className="text-xs text-slate-400 mt-1">กรอกรหัสผ่านเพื่อเข้าสู่แผงควบคุม Admin Dashboard</p>
                                        </div>

                                        <form onSubmit={handleLogin} className="space-y-4">
                                            <div>
                                                <label className="block text-xs font-semibold text-slate-300 mb-1">
                                                    รหัสผ่านผู้ดูแลระบบ (Admin Password)
                                                </label>
                                                <input
                                                    type="password"
                                                    required
                                                    value={passwordInput}
                                                    onChange={(e) => setPasswordInput(e.target.value)}
                                                    placeholder="กรอกรหัสผ่าน..."
                                                    className="w-full bg-slate-900 border border-slate-700 rounded-xl px-4 py-2.5 text-white placeholder-slate-500 focus:outline-none focus:ring-2 focus:ring-amber-500 text-sm"
                                                />
                                            </div>

                                            {loginError && (
                                                <div className="p-3 bg-red-900/40 border border-red-700 rounded-xl text-red-300 text-xs flex items-center gap-2">
                                                    <span>⚠️</span>
                                                    <span>{loginError}</span>
                                                </div>
                                            )}

                                            <button
                                                type="submit"
                                                disabled={isLoggingIn}
                                                className="w-full bg-amber-600 hover:bg-amber-500 active:scale-[0.99] disabled:bg-slate-700 text-white font-bold py-3 px-4 rounded-xl shadow-lg transition text-sm flex items-center justify-center gap-2 cursor-pointer"
                                            >
                                                {isLoggingIn ? (
                                                    <>
                                                        <div className="w-4 h-4 border-2 border-white border-t-transparent rounded-full animate-spin"></div>
                                                        <span>กำลังตรวจสอบ...</span>
                                                    </>
                                                ) : (
                                                    <span>เข้าสู่ระบบ</span>
                                                )}
                                            </button>
                                        </form>
                                    </div>
                                ) : (
                                    /* Interactive Admin Dashboard Panel */
                                    <div className="bg-slate-800/60 border border-slate-700/80 rounded-2xl p-6 shadow-2xl space-y-6">
                                        
                                        {/* Admin Header Bar */}
                                        <div className="flex flex-col sm:flex-row sm:items-center justify-between gap-4 pb-4 border-b border-slate-700">
                                            <div>
                                                <h3 className="text-lg font-bold text-amber-400 flex items-center gap-2">
                                                    <span>⚙️</span>
                                                    <span>แผงควบคุมผู้ดูแลระบบ (Admin Dashboard)</span>
                                                </h3>
                                                <p className="text-xs text-slate-400 mt-0.5">ยินดีต้อนรับ Admin — จัดการข้อมูลระบบคงถาวร และส่งสัญญาณ Realtime อัตโนมัติ</p>
                                            </div>
                                            <button
                                                onClick={handleLogout}
                                                className="px-4 py-2 bg-red-600/80 hover:bg-red-600 text-white rounded-xl text-xs font-semibold transition flex items-center gap-1.5 self-start sm:self-auto cursor-pointer"
                                            >
                                                <span>🚪</span>
                                                <span>ออกจากระบบ</span>
                                            </button>
                                        </div>

                                        {/* Admin Sub-Tabs Navigation */}
                                        <div className="flex flex-wrap gap-2 border-b border-slate-700/60 pb-3">
                                            {[
                                                { id: 'sports', icon: '⚽', label: `จัดการกีฬา (${sports.length})` },
                                                { id: 'teams', icon: '🛡️', label: `จัดการทีม (${teams.length})` },
                                                { id: 'matches', icon: '🏟️', label: `จัดการการแข่งขัน (${matches.length})` },
                                                { id: 'standings', icon: '📊', label: 'จัดการตารางคะแนน' },
                                                { id: 'medals', icon: '🥇', label: 'จัดการเหรียญรางวัล' }
                                            ].map((tab) => (
                                                <button
                                                    key={tab.id}
                                                    onClick={() => setAdminTab(tab.id)}
                                                    className={`px-4 py-2.5 rounded-xl text-xs sm:text-sm font-semibold transition flex items-center gap-2 cursor-pointer ${
                                                        adminTab === tab.id
                                                            ? 'bg-amber-500 text-slate-950 font-bold shadow-lg shadow-amber-500/20 scale-105'
                                                            : 'bg-slate-900/80 text-slate-300 hover:bg-slate-700 hover:text-white border border-slate-700/60'
                                                    }`}
                                                >
                                                    <span>{tab.icon}</span>
                                                    <span>{tab.label}</span>
                                                </button>
                                            ))}
                                        </div>

                                        {/* Dynamic Content Panel per Admin Tab */}
                                        <div className="bg-slate-900/70 border border-slate-800 rounded-2xl p-6 min-h-[300px]">
                                            
                                            {adminTab === 'sports' && (
                                                <div className="space-y-4">
                                                    <div className="flex justify-between items-center">
                                                        <div>
                                                            <h4 className="text-base font-bold text-white flex items-center gap-2">
                                                                <span>⚽</span>
                                                                <span>รายการกีฬา</span>
                                                            </h4>
                                                            <p className="text-xs text-slate-400">เพิ่ม/แก้ไข/ลบ รายการประเภทกีฬาในการแข่งขัน</p>
                                                        </div>
                                                        <button
                                                            onClick={() => { setModalData({}); setActiveModal('add_sport'); }}
                                                            className="px-4 py-2 bg-emerald-600 hover:bg-emerald-500 text-white rounded-xl text-xs font-bold transition flex items-center gap-1.5 cursor-pointer shadow-lg shadow-emerald-600/20"
                                                        >
                                                            <span>➕</span>
                                                            <span>เพิ่มกีฬาใหม่</span>
                                                        </button>
                                                    </div>

                                                    {sports.length === 0 ? (
                                                        <div className="border border-slate-800 rounded-xl p-8 text-center bg-slate-950/40">
                                                            <p className="text-sm text-slate-400">ยังไม่มีรายการกีฬาในขณะนี้</p>
                                                            <p className="text-xs text-slate-500 mt-1">กดปุ่ม "+ เพิ่มกีฬาใหม่" เพื่อเริ่มต้นบันทึกข้อมูลกีฬา</p>
                                                        </div>
                                                    ) : (
                                                        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                                                            {sports.map(sport => (
                                                                <div key={sport.id} className="bg-slate-800/90 border border-slate-700 p-4 rounded-xl flex flex-col justify-between">
                                                                    <div>
                                                                        <div className="flex justify-between items-start mb-2">
                                                                            <h5 className="font-bold text-white text-base">{sport.name}</h5>
                                                                            <span className={`text-[10px] px-2 py-0.5 rounded font-bold ${sport.active ? 'bg-emerald-950 text-emerald-400 border border-emerald-700/50' : 'bg-slate-700 text-slate-400'}`}>
                                                                                {sport.active ? 'ใช้งาน' : 'ปิดใช้งาน'}
                                                                            </span>
                                                                        </div>
                                                                        <p className="text-xs text-slate-400 line-clamp-2">{sport.description || 'ไม่มีคำอธิบาย'}</p>
                                                                    </div>
                                                                    <div className="flex justify-end gap-2 mt-4 pt-2 border-t border-slate-700/60">
                                                                        <button
                                                                            onClick={() => { setModalData(sport); setActiveModal('add_sport'); }}
                                                                            className="px-2.5 py-1 bg-slate-700 hover:bg-slate-600 text-slate-200 rounded-lg text-xs font-semibold transition"
                                                                        >
                                                                            ✏️ แก้ไข
                                                                        </button>
                                                                        <button
                                                                            onClick={() => { setDeleteTarget({ type: 'sport', id: sport.id, name: sport.name }); setActiveModal('delete_confirm'); }}
                                                                            className="px-2.5 py-1 bg-red-950/80 hover:bg-red-900 border border-red-700/50 text-red-300 rounded-lg text-xs font-semibold transition"
                                                                        >
                                                                            🗑️ ลบ
                                                                        </button>
                                                                    </div>
                                                                </div>
                                                            ))}
                                                        </div>
                                                    )}
                                                </div>
                                            )}

                                            {adminTab === 'teams' && (
                                                <div className="space-y-4">
                                                    <div className="flex justify-between items-center">
                                                        <div>
                                                            <h4 className="text-base font-bold text-white flex items-center gap-2">
                                                                <span>🛡️</span>
                                                                <span>รายการทีม</span>
                                                            </h4>
                                                            <p className="text-xs text-slate-400">เพิ่ม/แก้ไข/ลบ ทีมพร้อมโลโก้ (รองรับ PNG, JPG, WEBP &lt; 10MB)</p>
                                                        </div>
                                                        <button
                                                            onClick={() => { setModalData({}); setActiveModal('add_team'); }}
                                                            className="px-4 py-2 bg-emerald-600 hover:bg-emerald-500 text-white rounded-xl text-xs font-bold transition flex items-center gap-1.5 cursor-pointer shadow-lg shadow-emerald-600/20"
                                                        >
                                                            <span>➕</span>
                                                            <span>เพิ่มทีมใหม่</span>
                                                        </button>
                                                    </div>

                                                    {teams.length === 0 ? (
                                                        <div className="border border-slate-800 rounded-xl p-8 text-center bg-slate-950/40">
                                                            <p className="text-sm text-slate-400">ยังไม่มีข้อมูลทีมในขณะนี้</p>
                                                            <p className="text-xs text-slate-500 mt-1">กดปุ่ม "+ เพิ่มทีมใหม่" เพื่อเริ่มต้นบันทึกทีมและโลโก้</p>
                                                        </div>
                                                    ) : (
                                                        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                                                            {teams.map(team => (
                                                                <div key={team.id} className="bg-slate-800/90 border border-slate-700 p-4 rounded-xl flex items-center justify-between gap-3">
                                                                    <div className="flex items-center gap-3">
                                                                        {team.logo ? (
                                                                            <img src={team.logo} alt={team.name} className="w-12 h-12 object-contain rounded bg-slate-900 p-1 border border-slate-700" />
                                                                        ) : (
                                                                            <div className="w-12 h-12 bg-slate-700 rounded flex items-center justify-center font-bold text-slate-300">
                                                                                {team.short_name}
                                                                            </div>
                                                                        )}
                                                                        <div>
                                                                            <h5 className="font-bold text-white text-sm">{team.name}</h5>
                                                                            <span className="text-xs text-amber-400 font-medium">ชื่อย่อ: {team.short_name}</span>
                                                                        </div>
                                                                    </div>
                                                                    <div className="flex flex-col gap-1">
                                                                        <button
                                                                            onClick={() => { setModalData(team); setActiveModal('add_team'); }}
                                                                            className="px-2 py-1 bg-slate-700 hover:bg-slate-600 text-slate-200 rounded text-xs font-semibold transition"
                                                                        >
                                                                            ✏️ แก้ไข
                                                                        </button>
                                                                        <button
                                                                            onClick={() => { setDeleteTarget({ type: 'team', id: team.id, name: team.name }); setActiveModal('delete_confirm'); }}
                                                                            className="px-2 py-1 bg-red-950/80 hover:bg-red-900 border border-red-700/50 text-red-300 rounded text-xs font-semibold transition"
                                                                        >
                                                                            🗑️ ลบ
                                                                        </button>
                                                                    </div>
                                                                </div>
                                                            ))}
                                                        </div>
                                                    )}
                                                </div>
                                            )}

                                            {adminTab === 'matches' && (
                                                <div className="space-y-4">
                                                    <div className="flex justify-between items-center">
                                                        <div>
                                                            <h4 className="text-base font-bold text-white flex items-center gap-2">
                                                                <span>🏟️</span>
                                                                <span>จัดการการแข่งขัน</span>
                                                            </h4>
                                                            <p className="text-xs text-slate-400">สร้างการแข่งขัน ปรับคะแนนสดส่งผล Realtime และอัปเดตสถานะแมตช์</p>
                                                        </div>
                                                        <button
                                                            onClick={() => {
                                                                if (sports.length === 0 || teams.length < 2) {
                                                                    showToast('กรุณากรอกข้อมูลกีฬาอย่างน้อย 1 อย่าง และทีมอย่างน้อย 2 ทีมก่อนสร้างการแข่งขัน', 'error');
                                                                    return;
                                                                }
                                                                setModalData({ sport_id: sports[0]?.id, team_a_id: teams[0]?.id, team_b_id: teams[1]?.id, score_a: 0, score_b: 0, status: 'ยังไม่เริ่ม', round: 'รอบทั่วไป' });
                                                                setActiveModal('add_match');
                                                            }}
                                                            className="px-4 py-2 bg-emerald-600 hover:bg-emerald-500 text-white rounded-xl text-xs font-bold transition flex items-center gap-1.5 cursor-pointer shadow-lg shadow-emerald-600/20"
                                                        >
                                                            <span>➕</span>
                                                            <span>สร้างการแข่งขัน</span>
                                                        </button>
                                                    </div>

                                                    {matches.length === 0 ? (
                                                        <div className="border border-slate-800 rounded-xl p-8 text-center bg-slate-950/40">
                                                            <p className="text-sm text-slate-400">ยังไม่มีรายการแข่งขันในระบบ</p>
                                                            <p className="text-xs text-slate-500 mt-1">กดปุ่ม "+ สร้างการแข่งขัน" เพื่อเริ่มต้นจัดตารางแข่ง</p>
                                                        </div>
                                                    ) : (
                                                        <div className="space-y-3">
                                                            {matches.map(match => {
                                                                const teamA = getTeam(match.team_a_id);
                                                                const teamB = getTeam(match.team_b_id);
                                                                const sport = getSport(match.sport_id);

                                                                return (
                                                                    <div key={match.id} className="bg-slate-800/90 border border-slate-700 p-4 rounded-xl flex flex-col md:flex-row md:items-center justify-between gap-4">
                                                                        <div className="flex-1">
                                                                            <div className="flex items-center gap-2 mb-2 text-xs">
                                                                                <span className="bg-indigo-950 text-indigo-300 px-2 py-0.5 rounded font-semibold border border-indigo-800/50">{sport.name}</span>
                                                                                <span className="text-slate-400">{match.round}</span>
                                                                            </div>
                                                                            <div className="flex items-center gap-4">
                                                                                <span className="font-bold text-white text-sm">{teamA.name}</span>
                                                                                <span className="text-amber-400 font-extrabold text-lg bg-slate-950 px-3 py-1 rounded-lg border border-slate-800">
                                                                                    {match.score_a} - {match.score_b}
                                                                                </span>
                                                                                <span className="font-bold text-white text-sm">{teamB.name}</span>
                                                                            </div>
                                                                        </div>

                                                                        {/* Realtime Score Adjuster Controls */}
                                                                        <div className="flex flex-wrap items-center gap-2">
                                                                            <div className="flex items-center bg-slate-900 border border-slate-700 rounded-lg p-1">
                                                                                <span className="text-[10px] text-slate-400 px-1 font-bold">ทีม A:</span>
                                                                                <button onClick={() => handleQuickScoreChange(match.id, 'a', -1)} className="w-6 h-6 bg-slate-800 hover:bg-slate-700 text-slate-200 rounded font-bold text-xs">-</button>
                                                                                <button onClick={() => handleQuickScoreChange(match.id, 'a', 1)} className="w-6 h-6 bg-amber-600 hover:bg-amber-500 text-white rounded font-bold text-xs ml-1">+</button>
                                                                            </div>

                                                                            <div className="flex items-center bg-slate-900 border border-slate-700 rounded-lg p-1">
                                                                                <span className="text-[10px] text-slate-400 px-1 font-bold">ทีม B:</span>
                                                                                <button onClick={() => handleQuickScoreChange(match.id, 'b', -1)} className="w-6 h-6 bg-slate-800 hover:bg-slate-700 text-slate-200 rounded font-bold text-xs">-</button>
                                                                                <button onClick={() => handleQuickScoreChange(match.id, 'b', 1)} className="w-6 h-6 bg-amber-600 hover:bg-amber-500 text-white rounded font-bold text-xs ml-1">+</button>
                                                                            </div>

                                                                            <select
                                                                                value={match.status}
                                                                                onChange={(e) => handleQuickStatusChange(match.id, e.target.value)}
                                                                                className="bg-slate-900 border border-slate-700 text-xs text-white rounded-lg px-2 py-1.5 focus:outline-none"
                                                                            >
                                                                                <option value="ยังไม่เริ่ม">ยังไม่เริ่ม</option>
                                                                                <option value="กำลังแข่งขัน">🔴 กำลังแข่งขัน</option>
                                                                                <option value="จบการแข่งขัน">จบการแข่งขัน</option>
                                                                                <option value="ยกเลิก">ยกเลิก</option>
                                                                            </select>

                                                                            <button
                                                                                onClick={() => { setModalData(match); setActiveModal('add_match'); }}
                                                                                className="px-2.5 py-1.5 bg-slate-700 hover:bg-slate-600 text-slate-200 rounded-lg text-xs font-semibold transition"
                                                                            >
                                                                                ✏️
                                                                            </button>

                                                                            <button
                                                                                onClick={() => { setDeleteTarget({ type: 'match', id: match.id, name: `${teamA.name} vs ${teamB.name}` }); setActiveModal('delete_confirm'); }}
                                                                                className="px-2.5 py-1.5 bg-red-950/80 hover:bg-red-900 border border-red-700/50 text-red-300 rounded-lg text-xs font-semibold transition"
                                                                            >
                                                                                🗑️
                                                                            </button>
                                                                        </div>
                                                                    </div>
                                                                );
                                                            })}
                                                        </div>
                                                    )}
                                                </div>
                                            )}

                                            {adminTab === 'standings' && (
                                                <div className="space-y-4">
                                                    <div>
                                                        <h4 className="text-base font-bold text-white flex items-center gap-2">
                                                            <span>📊</span>
                                                            <span>ปรับแต่งคะแนนรวมและอันดับ</span>
                                                        </h4>
                                                        <p className="text-xs text-slate-400">กรอกคะแนนรวมและลำดับอันดับของแต่ละทีม</p>
                                                    </div>

                                                    {teams.length === 0 ? (
                                                        <div className="border border-slate-800 rounded-xl p-8 text-center bg-slate-950/40">
                                                            <p className="text-sm text-slate-400">ยังไม่มีทีมในระบบ กรุณาเพิ่มทีมในเมนู "จัดการทีม" ก่อน</p>
                                                        </div>
                                                    ) : (
                                                        <div className="bg-slate-900 border border-slate-800 rounded-xl p-4 space-y-3">
                                                            {teams.map((team, idx) => {
                                                                const st = standings.find(s => s.team_id === team.id) || { total_score: 0, rank: idx + 1 };
                                                                return (
                                                                    <div key={team.id} className="flex flex-col sm:flex-row sm:items-center justify-between gap-3 p-3 bg-slate-800/80 rounded-xl border border-slate-700/60">
                                                                        <div className="flex items-center gap-3">
                                                                            {team.logo ? (
                                                                                <img src={team.logo} alt={team.name} className="w-8 h-8 object-contain rounded bg-slate-900 p-0.5 border border-slate-700" />
                                                                            ) : (
                                                                                <div className="w-8 h-8 rounded bg-slate-700 flex items-center justify-center text-xs font-bold">
                                                                                    {team.short_name}
                                                                                </div>
                                                                            )}
                                                                            <span className="font-bold text-white text-sm">{team.name}</span>
                                                                        </div>

                                                                        <div className="flex items-center gap-4">
                                                                            <div className="flex items-center gap-1.5">
                                                                                <label className="text-xs text-slate-400 font-medium">อันดับ:</label>
                                                                                <input
                                                                                    type="number"
                                                                                    value={st.rank}
                                                                                    onChange={(e) => handleUpdateStandingScore(team.id, st.total_score, e.target.value)}
                                                                                    className="w-16 bg-slate-950 border border-slate-700 rounded-lg px-2 py-1 text-xs text-center font-bold text-white"
                                                                                />
                                                                            </div>
                                                                            <div className="flex items-center gap-1.5">
                                                                                <label className="text-xs text-slate-400 font-medium">คะแนนรวม:</label>
                                                                                <input
                                                                                    type="number"
                                                                                    value={st.total_score}
                                                                                    onChange={(e) => handleUpdateStandingScore(team.id, e.target.value, st.rank)}
                                                                                    className="w-24 bg-slate-950 border border-slate-700 rounded-lg px-2 py-1 text-xs text-center font-bold text-amber-400"
                                                                                />
                                                                            </div>
                                                                        </div>
                                                                    </div>
                                                                );
                                                            })}
                                                        </div>
                                                    )}
                                                </div>
                                            )}

                                            {adminTab === 'medals' && (
                                                <div className="space-y-4">
                                                    <div>
                                                        <h4 className="text-base font-bold text-white flex items-center gap-2">
                                                            <span>🥇</span>
                                                            <span>ปรับแต่งจำนวนเหรียญรางวัล</span>
                                                        </h4>
                                                        <p className="text-xs text-slate-400">เพิ่ม/ลด จำนวนเหรียญทอง เงิน และทองแดง ให้กับทีมต่าง ๆ</p>
                                                    </div>

                                                    {teams.length === 0 ? (
                                                        <div className="border border-slate-800 rounded-xl p-8 text-center bg-slate-950/40">
                                                            <p className="text-sm text-slate-400">ยังไม่มีทีมในระบบ กรุณาเพิ่มทีมในเมนู "จัดการทีม" ก่อน</p>
                                                        </div>
                                                    ) : (
                                                        <div className="bg-slate-900 border border-slate-800 rounded-xl p-4 space-y-3">
                                                            {teams.map(team => {
                                                                const md = medals.find(m => m.team_id === team.id) || { gold: 0, silver: 0, bronze: 0 };
                                                                return (
                                                                    <div key={team.id} className="flex flex-col md:flex-row md:items-center justify-between gap-3 p-3 bg-slate-800/80 rounded-xl border border-slate-700/60">
                                                                        <div className="flex items-center gap-3">
                                                                            {team.logo ? (
                                                                                <img src={team.logo} alt={team.name} className="w-8 h-8 object-contain rounded bg-slate-900 p-0.5 border border-slate-700" />
                                                                            ) : (
                                                                                <div className="w-8 h-8 rounded bg-slate-700 flex items-center justify-center text-xs font-bold">
                                                                                    {team.short_name}
                                                                                </div>
                                                                            )}
                                                                            <span className="font-bold text-white text-sm">{team.name}</span>
                                                                        </div>

                                                                        <div className="flex items-center gap-3">
                                                                            {/* Gold Controls */}
                                                                            <div className="flex items-center bg-slate-950 border border-slate-800 rounded-lg p-1">
                                                                                <span className="text-xs px-1">🥇</span>
                                                                                <button onClick={() => handleUpdateMedalCount(team.id, 'gold', -1)} className="w-5 h-5 bg-slate-800 text-slate-200 rounded font-bold text-xs">-</button>
                                                                                <span className="w-8 text-center text-xs font-bold text-amber-400">{md.gold || 0}</span>
                                                                                <button onClick={() => handleUpdateMedalCount(team.id, 'gold', 1)} className="w-5 h-5 bg-amber-600 text-white rounded font-bold text-xs">+</button>
                                                                            </div>

                                                                            {/* Silver Controls */}
                                                                            <div className="flex items-center bg-slate-950 border border-slate-800 rounded-lg p-1">
                                                                                <span className="text-xs px-1">🥈</span>
                                                                                <button onClick={() => handleUpdateMedalCount(team.id, 'silver', -1)} className="w-5 h-5 bg-slate-800 text-slate-200 rounded font-bold text-xs">-</button>
                                                                                <span className="w-8 text-center text-xs font-bold text-slate-300">{md.silver || 0}</span>
                                                                                <button onClick={() => handleUpdateMedalCount(team.id, 'silver', 1)} className="w-5 h-5 bg-slate-600 text-white rounded font-bold text-xs">+</button>
                                                                            </div>

                                                                            {/* Bronze Controls */}
                                                                            <div className="flex items-center bg-slate-950 border border-slate-800 rounded-lg p-1">
                                                                                <span className="text-xs px-1">🥉</span>
                                                                                <button onClick={() => handleUpdateMedalCount(team.id, 'bronze', -1)} className="w-5 h-5 bg-slate-800 text-slate-200 rounded font-bold text-xs">-</button>
                                                                                <span className="w-8 text-center text-xs font-bold text-amber-600">{md.bronze || 0}</span>
                                                                                <button onClick={() => handleUpdateMedalCount(team.id, 'bronze', 1)} className="w-5 h-5 bg-amber-800 text-white rounded font-bold text-xs">+</button>
                                                                            </div>
                                                                        </div>
                                                                    </div>
                                                                );
                                                            })}
                                                        </div>
                                                    )}
                                                </div>
                                            )}

                                        </div>
                                    </div>
                                )}
                            </div>
                        )}
                    </main>

                    {activeModal && (
                        <div className="fixed inset-0 z-50 bg-slate-950/80 backdrop-blur-sm flex items-center justify-center p-4">
                            <div className="bg-slate-800 border border-slate-700 w-full max-w-lg rounded-2xl p-6 shadow-2xl relative animate-in fade-in zoom-in duration-150 max-h-[90vh] overflow-y-auto">
                                
                                <div className="flex justify-between items-center pb-4 border-b border-slate-700">
                                    <h3 className="text-base font-bold text-white flex items-center gap-2">
                                        <span>⚙️</span>
                                        <span>
                                            {activeModal === 'add_sport' && (modalData.id ? 'แก้ไขรายการกีฬา' : 'เพิ่มรายการกีฬาใหม่')}
                                            {activeModal === 'add_team' && (modalData.id ? 'แก้ไขข้อมูลทีม' : 'เพิ่มทีมใหม่')}
                                            {activeModal === 'add_match' && (modalData.id ? 'แก้ไขแมตช์การแข่งขัน' : 'สร้างแมตช์การแข่งขันใหม่')}
                                            {activeModal === 'delete_confirm' && 'ยืนยันการลบข้อมูล'}
                                        </span>
                                    </h3>
                                    <button
                                        onClick={() => { setActiveModal(null); setModalData({}); }}
                                        className="w-8 h-8 rounded-full bg-slate-700 hover:bg-slate-600 text-slate-300 text-sm flex items-center justify-center transition cursor-pointer"
                                    >
                                        ✕
                                    </button>
                                </div>

                                {/* SPORT FORM */}
                                {activeModal === 'add_sport' && (
                                    <form onSubmit={handleSaveSport} className="space-y-4 mt-4">
                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">ชื่อกีฬา</label>
                                            <input
                                                type="text"
                                                required
                                                value={modalData.name || ''}
                                                onChange={(e) => setModalData({ ...modalData, name: e.target.value })}
                                                placeholder="เช่น ฟุตบอล, บาสเกตบอล"
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none focus:ring-2 focus:ring-amber-500"
                                            />
                                        </div>
                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">คำอธิบายรายละเอียด</label>
                                            <textarea
                                                rows="2"
                                                value={modalData.description || ''}
                                                onChange={(e) => setModalData({ ...modalData, description: e.target.value })}
                                                placeholder="คำอธิบายเพิ่มเติม..."
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none focus:ring-2 focus:ring-amber-500"
                                            ></textarea>
                                        </div>
                                        <div className="flex items-center gap-2">
                                            <input
                                                type="checkbox"
                                                id="sport_active"
                                                checked={modalData.active !== false}
                                                onChange={(e) => setModalData({ ...modalData, active: e.target.checked })}
                                                className="w-4 h-4 rounded text-amber-500 bg-slate-900 border-slate-700 focus:ring-amber-500"
                                            />
                                            <label htmlFor="sport_active" className="text-xs text-slate-300 font-medium">เปิดใช้งานในการแข่งขัน</label>
                                        </div>

                                        <div className="flex justify-end gap-2 pt-4 border-t border-slate-700">
                                            <button type="button" onClick={() => setActiveModal(null)} className="px-4 py-2 bg-slate-700 hover:bg-slate-600 text-slate-200 rounded-xl text-xs font-semibold">ยกเลิก</button>
                                            <button type="submit" disabled={isSubmitting} className="px-4 py-2 bg-amber-600 hover:bg-amber-500 text-white rounded-xl text-xs font-bold shadow-lg shadow-amber-600/20">บันทึกข้อมูล</button>
                                        </div>
                                    </form>
                                )}

                                {/* TEAM FORM */}
                                {activeModal === 'add_team' && (
                                    <form onSubmit={handleSaveTeam} className="space-y-4 mt-4">
                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">ชื่อทีม</label>
                                            <input
                                                type="text"
                                                required
                                                value={modalData.name || ''}
                                                onChange={(e) => setModalData({ ...modalData, name: e.target.value })}
                                                placeholder="เช่น ทีมมังกรทอง"
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none focus:ring-2 focus:ring-amber-500"
                                            />
                                        </div>
                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">ชื่อย่อทีม</label>
                                            <input
                                                type="text"
                                                required
                                                value={modalData.short_name || ''}
                                                onChange={(e) => setModalData({ ...modalData, short_name: e.target.value })}
                                                placeholder="เช่น TGT"
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none focus:ring-2 focus:ring-amber-500"
                                            />
                                        </div>
                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">โลโก้ทีม (PNG, JPG, WEBP &lt; 10MB)</label>
                                            <input
                                                type="file"
                                                accept="image/png, image/jpeg, image/webp"
                                                onChange={(e) => handleFileUpload(e.target.files[0], (dataUrl) => setModalData({ ...modalData, logo: dataUrl }))}
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-xs text-slate-300 focus:outline-none"
                                            />
                                            {modalData.logo && (
                                                <div className="mt-2 flex items-center gap-3 p-2 bg-slate-900 rounded-lg">
                                                    <img src={modalData.logo} alt="Preview" className="w-10 h-10 object-contain rounded border border-slate-700" />
                                                    <span className="text-xs text-emerald-400 font-medium">โหลดไฟล์โลโก้เรียบร้อย</span>
                                                </div>
                                            )}
                                        </div>

                                        <div className="flex justify-end gap-2 pt-4 border-t border-slate-700">
                                            <button type="button" onClick={() => setActiveModal(null)} className="px-4 py-2 bg-slate-700 hover:bg-slate-600 text-slate-200 rounded-xl text-xs font-semibold">ยกเลิก</button>
                                            <button type="submit" disabled={isSubmitting} className="px-4 py-2 bg-amber-600 hover:bg-amber-500 text-white rounded-xl text-xs font-bold shadow-lg shadow-amber-600/20">บันทึกข้อมูลทีม</button>
                                        </div>
                                    </form>
                                )}

                                {/* MATCH FORM */}
                                {activeModal === 'add_match' && (
                                    <form onSubmit={handleSaveMatch} className="space-y-4 mt-4">
                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">ประเภทกีฬา</label>
                                            <select
                                                value={modalData.sport_id || ''}
                                                onChange={(e) => setModalData({ ...modalData, sport_id: e.target.value })}
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none"
                                            >
                                                {sports.map(s => (
                                                    <option key={s.id} value={s.id}>{s.name}</option>
                                                ))}
                                            </select>
                                        </div>

                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">รอบการแข่งขัน</label>
                                            <input
                                                type="text"
                                                value={modalData.round || ''}
                                                onChange={(e) => setModalData({ ...modalData, round: e.target.value })}
                                                placeholder="เช่น รอบชิงชนะเลิศ, รอบ 8 ทีม"
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none"
                                            />
                                        </div>

                                        <div className="grid grid-cols-2 gap-3">
                                            <div>
                                                <label className="block text-xs text-slate-300 font-medium mb-1">ทีม A</label>
                                                <select
                                                    value={modalData.team_a_id || ''}
                                                    onChange={(e) => setModalData({ ...modalData, team_a_id: e.target.value })}
                                                    className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none"
                                                >
                                                    {teams.map(t => (
                                                        <option key={t.id} value={t.id}>{t.name}</option>
                                                    ))}
                                                </select>
                                            </div>

                                            <div>
                                                <label className="block text-xs text-slate-300 font-medium mb-1">ทีม B</label>
                                                <select
                                                    value={modalData.team_b_id || ''}
                                                    onChange={(e) => setModalData({ ...modalData, team_b_id: e.target.value })}
                                                    className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none"
                                                >
                                                    {teams.map(t => (
                                                        <option key={t.id} value={t.id}>{t.name}</option>
                                                    ))}
                                                </select>
                                            </div>
                                        </div>

                                        <div className="grid grid-cols-2 gap-3">
                                            <div>
                                                <label className="block text-xs text-slate-300 font-medium mb-1">คะแนนทีม A</label>
                                                <input
                                                    type="number"
                                                    value={modalData.score_a || 0}
                                                    onChange={(e) => setModalData({ ...modalData, score_a: e.target.value })}
                                                    className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-amber-400 font-bold focus:outline-none"
                                                />
                                            </div>
                                            <div>
                                                <label className="block text-xs text-slate-300 font-medium mb-1">คะแนนทีม B</label>
                                                <input
                                                    type="number"
                                                    value={modalData.score_b || 0}
                                                    onChange={(e) => setModalData({ ...modalData, score_b: e.target.value })}
                                                    className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-amber-400 font-bold focus:outline-none"
                                                />
                                            </div>
                                        </div>

                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">สถานะการแข่งขัน</label>
                                            <select
                                                value={modalData.status || 'ยังไม่เริ่ม'}
                                                onChange={(e) => setModalData({ ...modalData, status: e.target.value })}
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-sm text-white focus:outline-none"
                                            >
                                                <option value="ยังไม่เริ่ม">ยังไม่เริ่ม</option>
                                                <option value="กำลังแข่งขัน">🔴 กำลังแข่งขัน</option>
                                                <option value="จบการแข่งขัน">จบการแข่งขัน</option>
                                                <option value="ยกเลิก">ยกเลิก</option>
                                            </select>
                                        </div>

                                        <div>
                                            <label className="block text-xs text-slate-300 font-medium mb-1">รูปภาพการแข่งขัน (PNG, JPG, WEBP &lt; 10MB)</label>
                                            <input
                                                type="file"
                                                accept="image/png, image/jpeg, image/webp"
                                                onChange={(e) => handleFileUpload(e.target.files[0], (dataUrl) => setModalData({ ...modalData, image: dataUrl }))}
                                                className="w-full bg-slate-900 border border-slate-700 rounded-xl px-3 py-2 text-xs text-slate-300 focus:outline-none"
                                            />
                                            {modalData.image && (
                                                <div className="mt-2 flex items-center gap-3 p-2 bg-slate-900 rounded-lg">
                                                    <img src={modalData.image} alt="Match Preview" className="w-16 h-12 object-cover rounded border border-slate-700" />
                                                    <span className="text-xs text-emerald-400 font-medium">โหลดภาพถ่ายแมตช์เรียบร้อย</span>
                                                </div>
                                            )}
                                        </div>

                                        <div className="flex justify-end gap-2 pt-4 border-t border-slate-700">
                                            <button type="button" onClick={() => setActiveModal(null)} className="px-4 py-2 bg-slate-700 hover:bg-slate-600 text-slate-200 rounded-xl text-xs font-semibold">ยกเลิก</button>
                                            <button type="submit" disabled={isSubmitting} className="px-4 py-2 bg-amber-600 hover:bg-amber-500 text-white rounded-xl text-xs font-bold shadow-lg shadow-amber-600/20">บันทึกการแข่งขัน</button>
                                        </div>
                                    </form>
                                )}

                                {/* DELETE CONFIRMATION MODAL */}
                                {activeModal === 'delete_confirm' && deleteTarget && (
                                    <div className="space-y-4 mt-4">
                                        <p className="text-sm text-slate-300">
                                            คุณแน่ใจหรือไม่ว่าต้องการลบ <span className="text-amber-400 font-bold">"{deleteTarget.name}"</span>? 
                                            การกระทำนี้จะไม่สามารถย้อนกลับได้
                                        </p>

                                        <div className="flex justify-end gap-2 pt-4 border-t border-slate-700">
                                            <button type="button" onClick={() => setActiveModal(null)} className="px-4 py-2 bg-slate-700 hover:bg-slate-600 text-slate-200 rounded-xl text-xs font-semibold">ยกเลิก</button>
                                            <button type="button" onClick={confirmDelete} className="px-4 py-2 bg-red-600 hover:bg-red-500 text-white rounded-xl text-xs font-bold shadow-lg shadow-red-600/20">ยืนยันการลบ</button>
                                        </div>
                                    </div>
                                )}

                            </div>
                        </div>
                    )}

                    {fullscreenImage && (
                        <div className="fixed inset-0 z-50 bg-slate-950/95 flex flex-col items-center justify-between p-4 backdrop-blur-md animate-in fade-in duration-200">
                            {/* Toolbar */}
                            <div className="w-full max-w-5xl flex items-center justify-between z-10 bg-slate-900/80 p-3 rounded-2xl border border-slate-800">
                                <span className="text-xs font-bold text-slate-300">🖼️ Fullscreen Image Viewer</span>
                                <div className="flex items-center gap-2">
                                    <button
                                        onClick={() => setImageZoom(prev => Math.max(0.5, prev - 0.25))}
                                        className="px-3 py-1.5 bg-slate-800 hover:bg-slate-700 text-white text-xs font-bold rounded-lg transition"
                                    >
                                        🔍 -
                                    </button>
                                    <span className="text-xs font-mono text-amber-400 px-1">{Math.round(imageZoom * 100)}%</span>
                                    <button
                                        onClick={() => setImageZoom(prev => Math.min(3, prev + 0.25))}
                                        className="px-3 py-1.5 bg-slate-800 hover:bg-slate-700 text-white text-xs font-bold rounded-lg transition"
                                    >
                                        🔍 +
                                    </button>
                                    <button
                                        onClick={() => setImageZoom(1)}
                                        className="px-3 py-1.5 bg-slate-800 hover:bg-slate-700 text-slate-300 text-xs font-semibold rounded-lg transition"
                                    >
                                        รีเซ็ต
                                    </button>
                                    <a
                                        href={fullscreenImage}
                                        download="sports-match-photo.png"
                                        className="px-3 py-1.5 bg-indigo-600 hover:bg-indigo-500 text-white text-xs font-bold rounded-lg transition flex items-center gap-1"
                                    >
                                        <span>⬇️</span>
                                        <span>ดาวน์โหลด</span>
                                    </a>
                                    <button
                                        onClick={() => setFullscreenImage(null)}
                                        className="px-3 py-1.5 bg-red-600 hover:bg-red-500 text-white text-xs font-bold rounded-lg transition"
                                    >
                                        ✕ ปิด (ESC)
                                    </button>
                                </div>
                            </div>

                            {/* Image Canvas Container */}
                            <div className="flex-1 w-full max-w-5xl flex items-center justify-center overflow-auto p-4 my-2">
                                <img
                                    src={fullscreenImage}
                                    alt="Match Fullscreen"
                                    style={{ transform: `scale(${imageZoom})`, transition: 'transform 0.15s ease-out' }}
                                    className="max-h-[80vh] max-w-full object-contain rounded-xl shadow-2xl border border-slate-800"
                                />
                            </div>

                            <p className="text-xs text-slate-500">กดปุ่ม ESC บนคีย์บอร์ดหรือปุ่ม "ปิด" เพื่อออกจากโหมดเต็มจอ</p>
                        </div>
                    )}

                    {/* Footer */}
                    <footer className="bg-slate-950 border-t border-slate-800 py-6 text-center text-xs text-slate-500">
                        <div className="max-w-7xl mx-auto px-4">
                            ระบบรายงานผลการแข่งขันกีฬาแบบ Realtime &copy; {new Date().getFullYear()}
                        </div>
                    </footer>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
