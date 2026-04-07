<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Money Tracker Pro - Antigravity</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body { font-family: 'Inter', sans-serif; }
    </style>
</head>
<body class="bg-gray-50 min-h-screen p-4 md:p-8">

    <div class="max-w-6xl mx-auto">
        <header class="flex flex-col md:flex-row justify-between items-start md:items-center mb-8 gap-4">
            <div>
                <h1 class="text-3xl font-bold text-gray-800">Money Tracker Pro 📈</h1>
                <p class="text-gray-500">Visualisasi & Manajemen Keuangan Otomatis</p>
            </div>
            <div class="flex gap-2">
                <button onclick="exportToExcel()" class="bg-green-600 hover:bg-green-700 text-white px-4 py-2 rounded-xl font-semibold transition flex items-center gap-2">
                    <span>📊</span> Export
                </button>
                <button onclick="clearData()" class="bg-white border border-red-200 text-red-500 hover:bg-red-50 px-4 py-2 rounded-xl font-semibold transition">
                    Reset
                </button>
            </div>
        </header>

        <div class="grid grid-cols-1 lg:grid-cols-12 gap-8">
            
            <div class="lg:col-span-4 space-y-6">
                <div class="bg-white p-6 rounded-2xl shadow-sm border border-gray-100">
                    <h4 class="font-bold text-gray-700 mb-4 text-lg">Input Transaksi</h4>
                    <div class="space-y-4">
                        <input type="text" id="input-desc" placeholder="Keterangan (e.g. Sewa Kantor)" class="w-full p-3 border border-gray-200 rounded-xl outline-none focus:ring-2 focus:ring-blue-500">
                        <select id="input-cat" class="w-full p-3 border border-gray-200 rounded-xl outline-none">
                            <option value="REVENUE">Pendapatan (Revenue)</option>
                            <option value="COGS">HPP (Modal Barang)</option>
                            <option value="EXPENSE">Beban (Operasional)</option>
                        </select>
                        <input type="number" id="input-amount" placeholder="Jumlah Rp" class="w-full p-3 border border-gray-200 rounded-xl outline-none">
                        <button onclick="addEntry()" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-xl transition shadow-lg shadow-blue-100">Simpan Data</button>
                    </div>
                </div>

                <div class="bg-white p-6 rounded-2xl shadow-sm border border-gray-100">
                    <h4 class="font-bold text-gray-700 mb-4">Struktur Biaya</h4>
                    <canvas id="myChart" width="100" height="100"></canvas>
                </div>
            </div>

            <div class="lg:col-span-8 space-y-6">
                <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
                    <div class="bg-blue-600 p-6 rounded-2xl shadow-blue-100 shadow-xl text-white">
                        <p class="text-sm opacity-80">Pendapatan</p>
                        <h3 id="display-revenue" class="text-xl font-bold">Rp 0</h3>
                    </div>
                    <div class="bg-white p-6 rounded-2xl shadow-sm border border-gray-100">
                        <p class="text-sm text-gray-500 font-medium">Pengeluaran</p>
                        <h3 id="display-expense" class="text-xl font-bold text-red-500">Rp 0</h3>
                    </div>
                    <div class="bg-white p-6 rounded-2xl shadow-sm border border-gray-100">
                        <p class="text-sm text-gray-500 font-medium">Laba Bersih</p>
                        <h3 id="display-net" class="text-xl font-bold text-green-600">Rp 0</h3>
                    </div>
                </div>

                <div class="bg-white p-6 rounded-2xl shadow-sm border border-gray-100">
                    <div class="flex justify-between items-center mb-6 gap-4">
                        <h4 class="font-bold text-gray-700 hidden md:block">Riwayat Transaksi</h4>
                        <input type="text" id="search-input" onkeyup="updateUI()" placeholder="Cari transaksi..." class="flex-1 max-w-sm p-2 bg-gray-50 border border-gray-200 rounded-xl outline-none text-sm focus:bg-white">
                    </div>
                    
                    <div class="overflow-y-auto max-h-[400px]">
                        <table class="w-full text-left">
                            <thead class="sticky top-0 bg-white">
                                <tr class="text-gray-400 text-xs border-b">
                                    <th class="pb-3 font-medium">TANGGAL</th>
                                    <th class="pb-3 font-medium">KETERANGAN</th>
                                    <th class="pb-3 font-medium">KATEGORI</th>
                                    <th class="pb-3 font-medium text-right">JUMLAH</th>
                                </tr>
                            </thead>
                            <tbody id="transaction-list" class="divide-y divide-gray-50"></tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        let ledger = JSON.parse(localStorage.getItem('antigravity_data')) || [];
        let myChart;

        function addEntry() {
            const desc = document.getElementById('input-desc').value;
            const cat = document.getElementById('input-cat').value;
            const amount = parseFloat(document.getElementById('input-amount').value);
            const date = new Date().toLocaleDateString('id-ID');

            if (!desc || isNaN(amount)) return alert("Data tidak valid!");

            ledger.unshift({ date, desc, cat, amount }); // Unshift agar data terbaru di atas
            localStorage.setItem('antigravity_data', JSON.stringify(ledger));
            
            document.getElementById('input-desc').value = '';
            document.getElementById('input-amount').value = '';
            updateUI();
        }

        function updateUI() {
            const list = document.getElementById('transaction-list');
            const search = document.getElementById('search-input').value.toLowerCase();
            list.innerHTML = '';
            
            let rev = 0, hpp = 0, exp = 0;

            // Filter pencarian
            const filteredData = ledger.filter(t => t.desc.toLowerCase().includes(search));

            ledger.forEach(t => {
                if(t.cat === 'REVENUE') rev += t.amount;
                if(t.cat === 'COGS') hpp += t.amount;
                if(t.cat === 'EXPENSE') exp += t.amount;
            });

            filteredData.forEach(t => {
                list.innerHTML += `<tr class="text-sm text-gray-700 hover:bg-gray-50 transition">
                    <td class="py-4 text-gray-400 text-xs">${t.date}</td>
                    <td class="py-4 font-semibold">${t.desc}</td>
                    <td class="py-4"><span class="px-2 py-1 rounded-lg text-[10px] font-bold ${getBadgeColor(t.cat)}">${t.cat}</span></td>
                    <td class="py-4 text-right font-bold">${formatIDR(t.amount)}</td>
                </tr>`;
            });

            document.getElementById('display-revenue').innerText = formatIDR(rev);
            document.getElementById('display-expense').innerText = formatIDR(hpp + exp);
            document.getElementById('display-net').innerText = formatIDR(rev - (hpp + exp));
            
            updateChart(rev, hpp, exp);
        }

        function updateChart(rev, hpp, exp) {
            const ctx = document.getElementById('myChart').getContext('2d');
            if (myChart) myChart.destroy(); // Hapus grafik lama sebelum update

            myChart = new Chart(ctx, {
                type: 'doughnut',
                data: {
                    labels: ['Revenue', 'HPP', 'Beban'],
                    datasets: [{
                        data: [rev, hpp, exp],
                        backgroundColor: ['#2563eb', '#f97316', '#a855f7'],
                        borderWidth: 0,
                        hoverOffset: 10
                    }]
                },
                options: {
                    plugins: { legend: { position: 'bottom' } },
                    cutout: '70%'
                }
            });
        }

        function exportToExcel() {
            let csv = "Tanggal,Keterangan,Kategori,Jumlah\n";
            ledger.forEach(r => csv += `${r.date},${r.desc},${r.cat},${r.amount}\n`);
            const blob = new Blob([csv], { type: 'text/csv' });
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.setAttribute('href', url);
            a.setAttribute('download', 'Laporan_Keuangan.csv');
            a.click();
        }

        function clearData() {
            if(confirm("Hapus semua data?")) {
                ledger = [];
                localStorage.clear();
                updateUI();
            }
        }

        function getBadgeColor(cat) {
            if(cat === 'REVENUE') return 'bg-blue-100 text-blue-700';
            if(cat === 'COGS') return 'bg-orange-100 text-orange-700';
            return 'bg-purple-100 text-purple-700';
        }

        function formatIDR(num) {
            return new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 }).format(num);
        }

        // Jalankan UI pertama kali
        updateUI();
    </script>
</body>
</html># smart-money-tracker
