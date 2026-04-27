<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Nico Tapia - Performance Pro</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&display=swap');
        body { background-color: #fcfcf9; font-family: 'Inter', sans-serif; }
        .chart-container { position: relative; width: 100%; height: 260px; }
        .italic-black { font-style: italic; font-weight: 900; text-transform: uppercase; letter-spacing: -0.05em; }
        
        .rpe-input {
            @apply w-12 bg-white border-2 border-stone-100 text-center text-sm font-black text-orange-600 rounded-lg py-1 focus:outline-none focus:border-orange-500 transition-all;
        }
        
        .btn-register {
            @apply bg-stone-900 text-white text-[10px] font-black uppercase tracking-widest px-6 py-4 rounded-xl hover:bg-orange-600 transition-all shadow-lg active:scale-95;
        }

        .card-pro {
            @apply bg-white rounded-[2rem] border border-stone-200 p-6 flex flex-col md:flex-row items-center gap-6 shadow-sm mb-4 transition-all hover:shadow-md;
        }
    </style>
</head>
<body class="text-stone-800 antialiased">

    <div class="max-w-6xl mx-auto px-4 py-8">
        
        <header class="flex flex-col md:flex-row md:items-end justify-between mb-8 border-b-2 pb-8 border-stone-100">
            <div class="space-y-1">
                <div class="flex items-center gap-2">
                    <span class="bg-black text-white text-[9px] font-black px-2 py-0.5 rounded tracking-widest uppercase italic shadow-sm">COACH: NICO TAPIA</span>
                    <span class="text-orange-600 font-bold text-[10px] uppercase tracking-widest italic">• PLANIFICACIÓN PROFESIONAL</span>
                </div>
                <h1 class="text-5xl italic-black text-stone-900">
                    HYROX <span class="text-orange-600">S1</span> - PERFORMANCE
                </h1>
                <p class="text-stone-400 font-medium italic text-sm">Fuerza, Potencia y Seguimiento de Carga.</p>
            </div>
            <div id="status-badge" class="mt-6 md:mt-0 text-[10px] font-black text-stone-400 border-2 border-stone-200 rounded-full px-6 py-2 uppercase tracking-widest italic">
                SISTEMA CONECTADO
            </div>
        </header>

        <div class="grid grid-cols-1 lg:grid-cols-4 gap-6 mb-8">
            <div class="lg:col-span-3 bg-white rounded-[2.5rem] shadow-sm border border-stone-200 p-8">
                <div class="flex justify-between items-center mb-6">
                    <h2 class="text-[10px] font-black text-stone-400 uppercase tracking-widest italic">Análisis de Esfuerzo Percibido (RPE)</h2>
                    <span class="text-[9px] font-bold text-stone-400 uppercase italic">Escala de 1 a 10</span>
                </div>
                <div class="chart-container">
                    <canvas id="mainChart"></canvas>
                </div>
            </div>
            
            <div class="bg-stone-900 text-white rounded-[2.5rem] p-8 shadow-xl flex flex-col justify-between">
                <div>
                    <h3 class="text-orange-500 font-black uppercase text-[10px] tracking-widest mb-4 italic">Carga Semanal</h3>
                    <div class="text-5xl italic-black text-white" id="weekly-avg">--</div>
                    <div class="text-[9px] text-stone-500 uppercase font-black tracking-widest mt-1">Promedio RPE Real</div>
                </div>
                <button onclick="fetchWorkoutData()" class="text-[9px] font-black uppercase tracking-widest bg-stone-800 p-3 rounded-xl hover:bg-orange-600 transition-colors">Refrescar Datos</button>
            </div>
        </div>

        <div class="mb-4 flex items-center gap-4">
            <h3 class="text-xs font-black uppercase text-stone-400 tracking-widest italic">Sesión Planificada</h3>
            <div class="h-[1px] flex-1 bg-stone-100"></div>
        </div>

        <div id="workout-container" class="space-y-4">
            <div class="text-center py-20 text-stone-300 font-black italic uppercase tracking-widest">
                Sincronizando con Supabase...
            </div>
        </div>
    </div>

    <script>
        const SUPABASE_URL = 'https://shutiqoewzforhebaxsd.supabase.co';
        const SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNodXRpcW9ld3pmb3JoZWJheHNkIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzczMTA1MzAsImV4cCI6MjA5Mjg4NjUzMH0.SHRjH_oeirCQqXU7WIOsnBMadDKHXuUvaTMLjH7a5-A'; 
        const _supabase = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

        let performanceChart;

        async function fetchWorkoutData() {
            const container = document.getElementById('workout-container');
            
            try {
                // Consultamos Planificación y sus Ejercicios
                const { data: plan, error: planError } = await _supabase
                    .from('planificacion')
                    .select(`
                        id, series, repeticiones, descanso, notas_coach,
                        ejercicios ( nombre, grupo_muscular, categoria )
                    `);

                // Consultamos Seguimiento
                const { data: logs, error: logError } = await _supabase
                    .from('seguimiento')
                    .select('rpe_esfuerzo, id_sesion');

                if (planError) throw planError;

                if (!plan || plan.length === 0) {
                    container.innerHTML = `<div class="text-center py-20 text-stone-400 italic">No hay planificación cargada.</div>`;
                    return;
                }

                container.innerHTML = plan.map(item => {
                    const log = logs?.find(l => l.id_sesion === item.id);
                    return `
                    <div class="card-pro">
                        <div class="flex-1">
                            <div class="flex items-center gap-2 mb-2">
                                <span class="bg-orange-100 text-orange-600 text-[8px] font-black px-2 py-0.5 rounded uppercase tracking-widest italic">
                                    ${item.ejercicios.categoria || 'GENERAL'}
                                </span>
                            </div>
                            <h4 class="text-2xl italic-black text-stone-900">${item.ejercicios.nombre}</h4>
                            <p class="text-stone-400 text-[10px] font-bold uppercase italic mt-1">${item.notas_coach || ''}</p>
                        </div>
                        
                        <div class="grid grid-cols-3 gap-3 bg-stone-50 p-4 rounded-[1.5rem] border border-stone-100 w-full md:w-auto">
                            <div class="text-center">
                                <p class="text-[7px] font-black text-stone-400 uppercase mb-1">Series x Reps</p>
                                <p class="text-sm font-black italic text-stone-800">${item.series} x ${item.repeticiones}</p>
                            </div>
                            <div class="text-center border-x border-stone-200 px-4">
                                <p class="text-[7px] font-black text-stone-400 uppercase mb-1">Descanso</p>
                                <p class="text-sm font-black italic text-stone-800">${item.descanso}</p>
                            </div>
                            <div class="text-center">
                                <p class="text-[7px] font-black text-orange-600 uppercase mb-1">Tu RPE</p>
                                <input type="number" id="rpe-${item.id}" value="${log ? log.rpe_esfuerzo : ''}" 
                                       class="rpe-input">
                            </div>
                        </div>

                        <button onclick="saveLog(${item.id})" class="btn-register ${log ? 'bg-green-600' : ''}">
                            ${log ? '✓ GUARDADO' : 'REGISTRAR'}
                        </button>
                    </div>`;
                }).join('');

                updateMetrics(logs);

            } catch (err) {
                console.error(err);
                container.innerHTML = `<div class="p-10 text-red-500 font-bold text-center">ERROR DE CONEXIÓN: Verifica permisos en Supabase.</div>`;
            }
        }

        async function saveLog(planId) {
            const rpeVal = document.getElementById(`rpe-${planId}`).value;
            if(!rpeVal) return alert("Ingresa un valor RPE");

            const { error } = await _supabase
                .from('seguimiento')
                .upsert([
                    { id_sesion: planId, rpe_esfuerzo: parseInt(rpeVal), atleta_id: 1 }
                ], { onConflict: 'id_sesion' });

            if(error) alert("Error: " + error.message);
            else {
                fetchWorkoutData();
            }
        }

        function updateMetrics(logs) {
            if(!logs || logs.length === 0) return;
            const validLogs = logs.filter(l => l.rpe_esfuerzo > 0);
            const avg = (validLogs.reduce((a, b) => a + b.rpe_esfuerzo, 0) / validLogs.length).toFixed(1);
            document.getElementById('weekly-avg').innerText = avg;
            
            performanceChart.data.labels = validLogs.map((_, i) => `Eje ${i+1}`);
            performanceChart.data.datasets[0].data = validLogs.map(l => l.rpe_esfuerzo);
            performanceChart.update();
        }

        function initChart() {
            const ctx = document.getElementById('mainChart').getContext('2d');
            performanceChart = new Chart(ctx, {
                type: 'bar',
                data: {
                    labels: [],
                    datasets: [{
                        data: [],
                        backgroundColor: '#ea580c',
                        borderRadius: 8,
                        barThickness: 20
                    }]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: { legend: { display: false } },
                    scales: {
                        y: { beginAtZero: true, max: 10, ticks: { font: { weight: '900', size: 10 } } },
                        x: { grid: { display: false }, ticks: { font: { weight: '900', size: 9 } } }
                    }
                }
            });
        }

        document.addEventListener('DOMContentLoaded', () => {
            initChart();
            fetchWorkoutData();
        });
    </script>
</body>
</html>
