/
├── vercel.json
├── public/
│   └── index.html (Kode dashboard kamu yang sudah dimodifikasi)
└── api/
    ├── generate.js (Logika kirim prompt)
    └── status.js   (Logika cek status video)
    const axios = require('axios');

export default async function handler(req, res) {
    if (req.method !== 'POST') return res.status(405).send('Method Not Allowed');
    
    const { prompt } = req.body;
    try {
        const response = await axios.post('https://api.freepik.com/v1/ai/text-to-video', {
            prompt: prompt,
            model: 'video-high-res'
        }, {
            headers: { 
                'x-api-key': process.env.FREEPIK_API_KEY,
                'Content-Type': 'application/json' 
            }
        });
        res.status(200).json(response.data);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
}
const axios = require('axios');

export default async function handler(req, res) {
    const { id } = req.query;
    try {
        const response = await axios.get(`https://api.freepik.com/v1/ai/tasks/${id}`, {
            headers: { 'x-api-key': process.env.FREEPIK_API_KEY }
        });
        res.status(200).json(response.data);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
}
// Integrasi API Freepik ke Dashboard UniverseAI
async function processGeneration() {
    const promptValue = document.getElementById('motion-prompt').value.trim();
    if (!promptValue) {
        alert("Silahkan isi prompt terlebih dahulu!");
        return;
    }

    // Masuk ke tampilan Processing
    document.getElementById('sub-view-eva').classList.add('hidden-view');
    document.getElementById('sub-view-processing').classList.remove('hidden-view');
    
    try {
        // 1. Panggil API Backend kita
        const res = await fetch('/api/generate', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ prompt: promptValue })
        });
        
        const data = await res.json();
        if (!res.ok) throw new Error(data.error);

        const taskId = data.data.id;
        
        // 2. Mulai Cek Status (Polling)
        let percent = 10;
        const checkTimer = setInterval(async () => {
            const statusRes = await fetch(`/api/status?id=${taskId}`);
            const statusData = await statusRes.json();
            
            // Simulasi Progress Bar
            if (percent < 90) percent += 5;
            document.getElementById('progress-percent').innerText = percent + '% Rendering...';

            if (statusData.data.status === 'completed') {
                clearInterval(checkTimer);
                document.getElementById('progress-percent').innerText = '100% Selesai!';
                showResultsWithVideo(statusData.data.url);
            } else if (statusData.data.status === 'failed') {
                clearInterval(checkTimer);
                alert("AI Gagal memproses video.");
                switchDashboardTab('eva');
            }
        }, 4000); // Cek setiap 4 detik

    } catch (err) {
        alert("Sistem Error: " + err.message);
        switchDashboardTab('eva');
    }
}

// Fungsi menampilkan hasil video asli ke dalam Grid
function showResultsWithVideo(videoUrl) {
    document.getElementById('sub-view-processing').classList.add('hidden-view');
    document.getElementById('sub-view-results').classList.remove('hidden-view');
    
    const grid = document.getElementById('results-grid');
    grid.innerHTML = '';
    
    let ratioClass = selectedRatio === '1:1' ? "aspect-square" : (selectedRatio === '16:9' ? "aspect-video" : "aspect-[9/16]");

    // Kita tampilkan video yang dihasilkan API di kartu pertama
    grid.innerHTML += `
        <div class="glass-card overflow-hidden rounded-[2rem] group border-cyan-500/50" data-aos="fade-up">
            <div class="${ratioClass} bg-black flex flex-col items-center justify-center relative">
                <video src="${videoUrl}" autoplay loop muted class="w-full h-full object-cover"></video>
                <div class="absolute inset-0 bg-black/40 opacity-0 group-hover:opacity-100 transition-all flex items-center justify-center gap-4">
                    <a href="${videoUrl}" download target="_blank" class="w-10 h-10 rounded-full bg-cyan-500 text-white flex items-center justify-center text-xs">⬇</a>
                </div>
            </div>
            <div class="p-5 border-t border-white/5">
                <p class="text-[10px] font-bold text-cyan-400">FREELING ENGINE</p>
                <p class="text-[8px] text-gray-500 uppercase tracking-widest">${selectedRatio} • ULTRA HD</p>
            </div>
        </div>
    `;
}
{
  "rewrites": [
    { "source": "/api/(.*)", "destination": "/api/$1" },
    { "source": "/(.*)", "destination": "/public/$1" }
  ]
}
