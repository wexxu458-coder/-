// Improved Misbaha script: replaces existing <script> contents
// Key fixes:
// - compute SVG circumference with getTotalLength()
// - reuse a single AudioContext created on first user gesture
// - robust loadState merging with fallbacks for azkarList & stats
// - debounced saveState to reduce localStorage churn
// - ARIA and keyboard accessibility improvements

// Default Azkar Dataset
const defaultAzkar = [
    { id: 1, text: "سبحان الله وبحمده", category: "التسبيح", virtue: "حُطَّتْ خَطَايَاهُ وَإِنْ كَانَتْ مِثْلَ زَبَدِ الْبَحْرِ" },
    { id: 2, text: "الحمد لله", category: "التحميد", virtue: "تَمْلَأُ الْمِيزَانَ" },
    { id: 3, text: "الله أكبر", category: "التكبير", virtue: "من أحب الكلام إلى الله" },
    { id: 4, text: "لا إله إلا الله", category: "التوحيد", virtue: "أفضل ما قلت أنا والنبيون من قبلي" },
    { id: 5, text: "أستغفر الله وأتوب إليه", category: "الاستغفار", virtue: "سبب لفتح الرزق وتيسير الأمور" },
    { id: 6, text: "اللهم صلِّ وسلم على نبينا محمد", category: "الصلاة على النبي", virtue: "من صلى عليّ صلاة صلى الله عليه بها عشراً" },
    { id: 7, text: "لا حول ولا قوة إلا بالله", category: "الحوقلة", virtue: "كنز من كنوز الجنة" }
];

// Global State
let state = {
    currentZikrIndex: 0,
    count: 0,
    target: 33,
    completedRounds: 0,
    sessionTotal: 0,
    soundEnabled: true,
    vibrateEnabled: true,
    alertEnabled: true,
    darkMode: false,
    azkarList: [],
    stats: {}
};

let chartInstance = null;
let progressRing = null;
let ringCircumference = 0;
let audioCtx = null;
let saveTimeout = null;
let hasUserInteractedForAudio = false;

// Initialize App
document.addEventListener('DOMContentLoaded', () => {
    // prepare some DOM ARIA attributes & keyboard hooks
    const countDisplay = document.getElementById('count-display');
    const counterBtn = document.getElementById('counter-btn');
    if (countDisplay) countDisplay.setAttribute('aria-live', 'polite');
    if (counterBtn) {
        counterBtn.setAttribute('aria-label', 'زر التسبيح — اضغط أو اضغط على المسافة لزيادة');
        counterBtn.addEventListener('keydown', (e) => {
            if (e.code === 'Space' || e.key === ' ' || e.key === 'Enter') {
                e.preventDefault();
                incrementCounter();
            }
        });
    }

    loadState();
    setupTheme();
    renderZikrPills();
    renderAzkarList();
    // Setup progress ring
    progressRing = document.getElementById('progress-ring');
    if (progressRing && typeof progressRing.getTotalLength === 'function') {
        ringCircumference = progressRing.getTotalLength();
        progressRing.style.strokeDasharray = ringCircumference;
        progressRing.style.strokeDashoffset = ringCircumference;
    } else {
        // fallback to reasonable constant (kept for older browsers)
        ringCircumference = 791;
        if (progressRing) {
            progressRing.style.strokeDasharray = ringCircumference;
            progressRing.style.strokeDashoffset = ringCircumference;
        }
    }
    updateCounterUI();
    updateStatsUI();
    initChart();

    // Resume/create audio context on first user gesture (required on some browsers)
    const resumeAudioOnFirstGesture = () => {
        if (!audioCtx) {
            try {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                // Keep it suspended until needed; resume on first sound play as required
            } catch (e) {
                audioCtx = null;
            }
        }
        hasUserInteractedForAudio = true;
        window.removeEventListener('pointerdown', resumeAudioOnFirstGesture);
    };
    window.addEventListener('pointerdown', resumeAudioOnFirstGesture, { once: true });

    // Keyboard shortcut for global spacebar while on counter tab
    document.addEventListener('keydown', (e) => {
        if (e.code === 'Space' && document.getElementById('tab-counter').classList.contains('active')) {
            e.preventDefault();
            incrementCounter();
        }
    });
});

// Debounced saveState to reduce write frequency
function saveState() {
    clearTimeout(saveTimeout);
    saveTimeout = setTimeout(() => {
        try {
            localStorage.setItem('misbaha_state_v1', JSON.stringify(state));
        } catch (e) {
            // ignore storage errors
        }
    }, 120);
}

// Local Storage State Loader (robust merge with defaults)
function loadState() {
    const savedState = localStorage.getItem('misbaha_state_v1');
    if (savedState) {
        try {
            const parsed = JSON.parse(savedState);
            // merge top-level but ensure azkarList and stats have fallbacks
            state = { ...state, ...parsed };
            state.azkarList = (parsed.azkarList && parsed.azkarList.length) ? parsed.azkarList : defaultAzkar.slice();
            state.stats = parsed.stats || {};
        } catch (e) {
            state.azkarList = defaultAzkar.slice();
        }
    } else {
        state.azkarList = defaultAzkar.slice();
    }

    // Sync UI toggles if they exist
    const soundToggle = document.getElementById('setting-sound-toggle');
    const vibrateToggle = document.getElementById('setting-vibrate-toggle');
    if (soundToggle) soundToggle.checked = !!state.soundEnabled;
    if (vibrateToggle) vibrateToggle.checked = !!state.vibrateEnabled;
    updateSoundIcon();
    updateVibrateIcon();
}

// Theme helpers (unchanged logic but preserve state)
function setupTheme() {
    if (state.darkMode || (!('darkMode' in state) && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
        document.documentElement.classList.add('dark');
        document.documentElement.classList.remove('light');
        state.darkMode = true;
    } else {
        document.documentElement.classList.remove('dark');
        document.documentElement.classList.add('light');
        state.darkMode = false;
    }
    updateThemeIcon();
}

function toggleDarkMode() {
    state.darkMode = !state.darkMode;
    if (state.darkMode) {
        document.documentElement.classList.add('dark');
        document.documentElement.classList.remove('light');
    } else {
        document.documentElement.classList.remove('dark');
        document.documentElement.classList.add('light');
    }
    updateThemeIcon();
    saveState();
}

function updateThemeIcon() {
    const icon = document.getElementById('theme-icon');
    if (!icon) return;
    if (state.darkMode) {
        icon.className = 'fa-solid fa-sun text-amber-400';
    } else {
        icon.className = 'fa-solid fa-moon text-slate-600';
    }
}

// Audio helpers using a single AudioContext
function getAudioCtx() {
    if (!audioCtx) {
        try {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        } catch (e) {
            audioCtx = null;
        }
    }
    // try to resume on demand (some browsers require user gesture)
    if (audioCtx && audioCtx.state === 'suspended' && hasUserInteractedForAudio) {
        audioCtx.resume().catch(() => {});
    }
    return audioCtx;
}

function playClickSound() {
    if (!state.soundEnabled) return;
    const ctx = getAudioCtx();
    if (!ctx) return;
    try {
        const osc = ctx.createOscillator();
        const gain = ctx.createGain();
        osc.type = 'sine';
        const now = ctx.currentTime;
        osc.frequency.setValueAtTime(500, now);
        osc.frequency.exponentialRampToValueAtTime(150, now + 0.04);
        gain.gain.setValueAtTime(0.12, now);
        gain.gain.exponentialRampToValueAtTime(0.01, now + 0.04);
        osc.connect(gain);
        gain.connect(ctx.destination);
        osc.start(now);
        osc.stop(now + 0.05);
    } catch (e) {
        // silent fail
    }
}

function playSuccessSound() {
    if (!state.soundEnabled) return;
    const ctx = getAudioCtx();
    if (!ctx) return;
    try {
        const notes = [523.25, 659.25, 783.99, 1046.50];
        const now = ctx.currentTime;
        notes.forEach((freq, idx) => {
            const osc = ctx.createOscillator();
            const gain = ctx.createGain();
            osc.type = 'triangle';
            const startAt = now + idx * 0.07;
            osc.frequency.setValueAtTime(freq, startAt);
            gain.gain.setValueAtTime(0.18, startAt);
            gain.gain.exponentialRampToValueAtTime(0.001, startAt + 0.4);
            osc.connect(gain);
            gain.connect(ctx.destination);
            osc.start(startAt);
            osc.stop(startAt + 0.42);
        });
    } catch (e) {}
}

function triggerVibration(pattern = [30]) {
    if (state.vibrateEnabled && navigator.vibrate) {
        try { navigator.vibrate(pattern); } catch (e) {}
    }
}

function toggleSound() {
    state.soundEnabled = !state.soundEnabled;
    const el = document.getElementById('setting-sound-toggle');
    if (el) el.checked = state.soundEnabled;
    updateSoundIcon();
    saveState();
}

function updateSoundIcon() {
    const icon = document.getElementById('sound-icon');
    if (!icon) return;
    if (state.soundEnabled) {
        icon.className = 'fa-solid fa-volume-high text-brand-600 dark:text-brand-400';
    } else {
        icon.className = 'fa-solid fa-volume-xmark text-slate-400';
    }
}

function toggleVibrate() {
    state.vibrateEnabled = !state.vibrateEnabled;
    const el = document.getElementById('setting-vibrate-toggle');
    if (el) el.checked = state.vibrateEnabled;
    updateVibrateIcon();
    saveState();
}

function updateVibrateIcon() {
    const icon = document.getElementById('vibrate-icon');
    if (!icon) return;
    if (state.vibrateEnabled) {
        icon.className = 'fa-solid fa-mobile-screen-button text-brand-600 dark:text-brand-400';
    } else {
        icon.className = 'fa-solid fa-mobile-xmark text-slate-400';
    }
}

// Tab switching (unchanged)
function switchTab(tabId) {
    document.querySelectorAll('.tab-content').forEach(el => el.classList.remove('active'));
    document.querySelectorAll('.nav-btn').forEach(btn => {
        btn.classList.remove('text-brand-600', 'dark:text-brand-400', 'font-bold');
        btn.classList.add('text-slate-400', 'font-medium');
    });

    const activeTab = document.getElementById(`tab-${tabId}`);
    const activeNav = document.getElementById(`nav-${tabId}`);

    if (activeTab) activeTab.classList.add('active');
    if (activeNav) {
        activeNav.classList.remove('text-slate-400', 'font-medium');
        activeNav.classList.add('text-brand-600', 'dark:text-brand-400', 'font-bold');
    }

    if (tabId === 'stats') {
        updateStatsUI();
        renderChart();
    }
}

// Counter core
function incrementCounter() {
    state.count++;
    state.sessionTotal++;

    const currentZikr = state.azkarList[state.currentZikrIndex] || state.azkarList[0];
    const zikrId = currentZikr.id;
    state.stats[zikrId] = (state.stats[zikrId] || 0) + 1;

    playClickSound();
    triggerVibration([25]);

    // Target reached handling
    if (state.target > 0 && state.count >= state.target) {
        state.completedRounds++;
        playSuccessSound();
        triggerVibration([100, 50, 100]);

        const alertEnabledEl = document.getElementById('setting-alert-toggle');
        const alertEnabled = alertEnabledEl ? alertEnabledEl.checked : state.alertEnabled;
        if (alertEnabled && typeof confetti === 'function') {
            confetti({ particleCount: 70, spread: 60, origin: { y: 0.7 } });
        }
        state.count = 0;
    }

    updateCounterUI();
    saveState();
}

function decrementCounter() {
    if (state.count > 0) {
        state.count--;
        state.sessionTotal = Math.max(0, state.sessionTotal - 1);
        const currentZikr = state.azkarList[state.currentZikrIndex] || state.azkarList[0];
        const zikrId = currentZikr.id;
        if (state.stats[zikrId] && state.stats[zikrId] > 0) {
            state.stats[zikrId]--;
        }
        updateCounterUI();
        saveState();
    }
}

function resetCurrentCounter() {
    if (confirm("هل تريد تصفير العداد للذكر الحالي؟")) {
        state.count = 0;
        updateCounterUI();
        saveState();
    }
}

function switchTargetQuick() {
    const targets = [33, 100, 1000, 0];
    let nextIndex = (targets.indexOf(state.target) + 1) % targets.length;
    setTarget(targets[nextIndex]);
}

function setTarget(newTarget) {
    state.target = newTarget;
    state.count = 0;
    updateCounterUI();
    saveState();
}

function selectZikr(index) {
    state.currentZikrIndex = index;
    state.count = 0;
    updateCounterUI();
    renderZikrPills();
    switchTab('counter');
    saveState();
}

// UI updates & rendering
function updateCounterUI() {
    const currentZikr = state.azkarList[state.currentZikrIndex] || state.azkarList[0];

    document.getElementById('current-zikr-text').textContent = currentZikr.text;
    document.getElementById('current-zikr-category').textContent = currentZikr.category || 'ذكر';
    document.getElementById('current-zikr-meaning').textContent = currentZikr.virtue ? `فضلها: ${currentZikr.virtue}` : '';

    document.getElementById('count-display').textContent = state.count;
    document.getElementById('target-display').textContent = state.target > 0 ? state.target : 'مفتوح';
    document.getElementById('rounds-display').textContent = state.completedRounds;
    document.getElementById('session-total-display').textContent = state.sessionTotal;

    // Update SVG progress ring using computed circumference
    if (progressRing && ringCircumference > 0) {
        if (state.target > 0) {
            const progress = Math.min(1, state.count / state.target);
            const offset = ringCircumference - (progress * ringCircumference);
            progressRing.style.strokeDashoffset = offset;
        } else {
            // open target: show full circle (or you could animate)
            progressRing.style.strokeDashoffset = 0;
        }
    }

    // Update target buttons' visual state
    document.querySelectorAll('.target-opt-btn').forEach(btn => {
        const val = parseInt(btn.textContent) || 0;
        const btnIsOpen = btn.textContent.trim() === 'مفتوح';
        if ((val === state.target) || (btnIsOpen && state.target === 0)) {
            btn.className = "target-opt-btn py-2.5 rounded-xl border border-brand-500 bg-brand-50 dark:bg-brand-950/40 text-brand-600 dark:text-brand-400 font-bold text-xs transition";
        } else {
            btn.className = "target-opt-btn py-2.5 rounded-xl border border-slate-200 dark:border-slate-700 text-slate-600 dark:text-slate-300 font-bold text-xs hover:border-brand-500 transition";
        }
    });
}

function renderZikrPills() {
    const container = document.getElementById('zikr-pills');
    if (!container) return;
    container.innerHTML = '';

    state.azkarList.forEach((zikr, idx) => {
        const isActive = idx === state.currentZikrIndex;
        const pill = document.createElement('button');
        pill.onclick = () => selectZikr(idx);
        pill.className = `px-4 py-2 rounded-2xl whitespace-nowrap text-xs font-bold transition flex items-center gap-2 border ${
            isActive
                ? 'bg-brand-600 text-white border-brand-600 shadow-sm shadow-brand-500/30'
                : 'bg-white dark:bg-slate-900 text-slate-600 dark:text-slate-300 border-slate-200/80 dark:border-slate-800 hover:border-brand-500'
        }`;
        pill.innerHTML = `<span>${zikr.text}</span>`;
        pill.setAttribute('aria-pressed', isActive ? 'true' : 'false');
        container.appendChild(pill);
    });
}

function renderAzkarList() {
    const container = document.getElementById('azkar-cards-container');
    if (!container) return;
    container.innerHTML = '';

    state.azkarList.forEach((zikr, idx) => {
        const card = document.createElement('div');
        card.className = "bg-white dark:bg-slate-900 rounded-2xl p-4 border border-slate-200/80 dark:border-slate-800 shadow-sm flex flex-col justify-between hover:border-brand-500/50 transition";
        card.innerHTML = `
            <div>
                <div class="flex justify-between items-start mb-2">
                    <span class="px-2.5 py-0.5 bg-slate-100 dark:bg-slate-800 text-slate-500 text-[10px] font-bold rounded-lg">${zikr.category || 'ذكر'}</span>
                    <span class="text-xs font-semibold text-brand-600 dark:text-brand-400">مرات التسبيح: ${state.stats[zikr.id] || 0}</span>
                </div>
                <h3 class="font-bold text-base text-slate-800 dark:text-slate-100 mb-1">${zikr.text}</h3>
                ${zikr.virtue ? `<p class="text-xs text-slate-400 leading-relaxed">${zikr.virtue}</p>` : ''}
            </div>
            <div class="mt-4 flex gap-2">
                <button onclick="selectZikr(${idx})" class="flex-1 py-2 bg-brand-50 dark:bg-brand-950/40 text-brand-600 dark:text-brand-300 rounded-xl text-xs font-bold hover:bg-brand-100 transition" aria-label="اختيار ${escapeHtml(zikr.text)} للتسبيح">
                    اختيار للتسبيح
                </button>
            </div>
        `;
        container.appendChild(card);
    });
}

// simple helper to avoid inserting raw HTML into ARIA labels
function escapeHtml(str) {
    return String(str).replace(/[&<>"']/g, (m) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[m]));
}

// Modal and custom zikr
function openCustomZikrModal() {
    document.getElementById('custom-zikr-modal').classList.remove('hidden');
}
function closeCustomZikrModal() {
    document.getElementById('custom-zikr-modal').classList.add('hidden');
}
function saveCustomZikr() {
    const textInputEl = document.getElementById('custom-zikr-input');
    const meaningInputEl = document.getElementById('custom-meaning-input');
    if (!textInputEl) return;
    const textInput = textInputEl.value.trim();
    const meaningInput = (meaningInputEl && meaningInputEl.value) ? meaningInputEl.value.trim() : '';

    if (!textInput) {
        alert("الرجاء كتابة نص الذكر");
        return;
    }

    const newZikr = {
        id: Date.now(),
        text: textInput,
        category: "مخصص",
        virtue: meaningInput || "ذكر مخصص من قبل المستخدم"
    };

    state.azkarList.push(newZikr);
    saveState();
    renderZikrPills();
    renderAzkarList();
    closeCustomZikrModal();

    if (textInputEl) textInputEl.value = '';
    if (meaningInputEl) meaningInputEl.value = '';

    selectZikr(state.azkarList.length - 1);
}

// Stats & Chart
function updateStatsUI() {
    let totalAll = 0;
    let topCount = 0;
    let topZikrName = "لا يوجد";

    state.azkarList.forEach(z => {
        const cnt = state.stats[z.id] || 0;
        totalAll += cnt;
        if (cnt > topCount) {
            topCount = cnt;
            topZikrName = z.text;
        }
    });

    document.getElementById('stat-total-all').textContent = totalAll;
    document.getElementById('stat-total-today').textContent = state.sessionTotal;
    document.getElementById('stat-total-rounds').textContent = state.completedRounds;
    document.getElementById('stat-top-zikr').textContent = topZikrName;
}

function initChart() {
    const canvas = document.getElementById('azkarChart');
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    chartInstance = new Chart(ctx, {
        type: 'doughnut',
        data: {
            labels: [],
            datasets: [{
                data: [],
                backgroundColor: [
                    '#10b981', '#0ea5e9', '#f59e0b', '#8b5cf6', '#ec4899', '#14b8a6', '#6366f1'
                ],
                borderWidth: 0
            }]
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
                legend: {
                    position: 'bottom',
                    labels: {
                        font: { family: 'Tajawal', size: 11 },
                        boxWidth: 12
                    }
                }
            },
            cutout: '70%'
        }
    });
}

function renderChart() {
    if (!chartInstance) return;

    const labels = [];
    const data = [];

    state.azkarList.forEach(z => {
        const count = state.stats[z.id] || 0;
        if (count > 0) {
            labels.push(z.text.length > 20 ? z.text.substring(0, 20) + '...' : z.text);
            data.push(count);
        }
    });

    if (data.length === 0) {
        chartInstance.data.labels = ['لا توجد تسبيحات بعد'];
        chartInstance.data.datasets[0].data = [1];
        chartInstance.data.datasets[0].backgroundColor = ['#e2e8f0'];
    } else {
        chartInstance.data.labels = labels;
        chartInstance.data.datasets[0].data = data;
        chartInstance.data.datasets[0].backgroundColor = [
            '#10b981', '#0ea5e9', '#f59e0b', '#8b5cf6', '#ec4899', '#14b8a6', '#6366f1'
        ];
    }
    chartInstance.update();
}

// Reset all data
function resetAllData() {
    if (confirm("هل أنت تأكد من استعادة الضبط وتصفير كافة الإحصائيات والأذكار؟")) {
        localStorage.removeItem('misbaha_state_v1');
        location.reload();
    }
}
