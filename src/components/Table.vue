<script setup>
import { ref, onMounted, computed } from 'vue'

const props = defineProps({
    initialTicker: { type: String, default: 'MSFT' },
    initialAssumed: { type: String, default: 'USD' },
    initialConverte: { type: String, default: 'IDR' },
})


const ticker = ref(props.initialTicker)
const assumed = ref(props.initialAssumed)
const converte = ref(props.initialConverte)

const companyMap = [
  { "company_name": "Apple Inc.", "ticker_symbol": "AAPL" },
  { "company_name": "Microsoft Corporation", "ticker_symbol": "MSFT" },
  { "company_name": "Amazon.com Inc.", "ticker_symbol": "AMZN" },
  { "company_name": "Alphabet Inc. (Google)", "ticker_symbol": "GOOGL" },
  { "company_name": "Meta Platforms Inc. (Facebook)", "ticker_symbol": "META" },
  { "company_name": "NVIDIA Corporation", "ticker_symbol": "NVDA" },
  { "company_name": "Tesla Inc.", "ticker_symbol": "TSLA" },
  { "company_name": "Berkshire Hathaway Inc.", "ticker_symbol": "BRK.B" },
  { "company_name": "Visa Inc.", "ticker_symbol": "V" },
  { "company_name": "Netflix Inc.", "ticker_symbol": "NFLX" }
]


const companyName = computed(() => {
  const found = companyMap.find(c => c.ticker_symbol === ticker.value)
  return found ? found.company_name : ''
})

const exchangeRates = [
    { country: 'United States', id: 'USD', exchange_rate_per_usd: 1.00 },
    { country: 'Indonesia', id: 'IDR', exchange_rate_per_usd: 15000.00 },
    { country: 'United Arab Emirates', id: 'AED', exchange_rate_per_usd: 3.67 },
    { country: 'Saudi Arabia', id: 'SAR', exchange_rate_per_usd: 3.75 },
    { country: 'Egypt', id: 'EGP', exchange_rate_per_usd: 30.90 },
    { country: 'Eurozone', id: 'EUR', exchange_rate_per_usd: 0.92 },
    { country: 'Japan', id: 'JPY', exchange_rate_per_usd: 140.00 },
    { country: 'China', id: 'CNY', exchange_rate_per_usd: 7.20 },
    { country: 'United Kingdom', id: 'GBP', exchange_rate_per_usd: 0.79 },
    { country: 'India', id: 'INR', exchange_rate_per_usd: 82.00 }
]

const getRate = (id) => {
    const r = exchangeRates.find(x => x.id === id)
    return r ? Number(r.exchange_rate_per_usd) : 1
}

const conversionFactor = computed(() => {
    const rateAssumed = getRate(assumed.value || 'USD')
    const rateConverte = getRate(converte.value || 'USD')
    return rateConverte / rateAssumed
})

const convertedTimeSeries = computed(() =>
    timeSeries.value.map(row => {
        const f = conversionFactor.value
        const toNum = v => {
            if (v == null) return 0
            return Number(String(v).replace(/,/g, ''))
        }
        return {
            date: row.date,
            open: (toNum(row.open) * f).toFixed(2),
            high: (toNum(row.high) * f).toFixed(2),
            low: (toNum(row.low) * f).toFixed(2),
            close: (toNum(row.close) * f).toFixed(2),
            volume: Math.round(toNum(row.volume) * f)
        }
    })
)

const apiKey = 'U2A8C3EMLYAEA3XR'
const loading = ref(false)
const error = ref('')
const timeSeries = ref([])
const meta = ref({})

async function fetchData() {
    error.value = ''
    timeSeries.value = []
    meta.value = {}

    if (!ticker.value) {
        error.value = 'Ticker is required.'
        return
    }

    loading.value = true
    try {
        const url = `https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol=${encodeURIComponent(
            ticker.value
        )}&apikey=${encodeURIComponent(apiKey)}`

        const res = await fetch(url)
        if (!res.ok) throw new Error(`HTTP ${res.status}`)
        const data = await res.json()

        if (data['Error Message']) {
            throw new Error(data['Error Message'])
        }
        if (data['Note']) {
            throw new Error(data['Note'])
        }

        const metaData = data['Meta Data'] || {}
        const series = data['Time Series (Daily)'] || {}

        const entries = Object.keys(series).map((date) => {
            const v = series[date]
            return {
                date,
                open: v['1. open'],
                high: v['2. high'],
                low: v['3. low'],
                close: v['4. close'],
                volume: v['5. volume']
            }
        })

        entries.sort((a, b) => (a.date < b.date ? 1 : -1))

        meta.value = metaData
        timeSeries.value = entries
    } catch (e) {
        error.value = e && e.message ? e.message : String(e)
    } finally {
        loading.value = false
    }
}

</script>

<template>
    <div class="av-table">
        <div class="controls">
            <label>
                Ticker
                <select v-model="ticker">
                    <option v-for="item in companyMap" :key="item.ticker_symbol" :value="item.ticker_symbol">
                        {{ item.company_name }}
                    </option>
                </select>
            </label>

            <label>
                Assumed
                <select v-model="assumed">
                    <option v-for="rate in exchangeRates" :key="rate.id" :value="rate.id">
                        {{ rate.id }}
                    </option>
                </select>
            </label>

            <label>
                Converte
                <select v-model="converte">
                     <option v-for="rate in exchangeRates" :key="rate.id" :value="rate.id">
                        {{ rate.id }}
                    </option>
                </select>
            </label>

            <button @click="fetchData" :disabled="loading">Fetch</button>
        </div>

        <div class="status">
            <span v-if="loading">Loading…</span>
            <span v-if="error" class="error">{{ error }}</span>
        </div>

        <div>
            <h2>Display Data</h2>
            <p>Assumed Currency : {{ assumed }}</p>
            <p>Converte Currency : {{ converte }}</p>
            <p>Company : {{ companyName }}</p>
            <p>Symbol : {{ ticker }}</p>
        </div>

        <template v-if="timeSeries.length">
            <h3>Original Price</h3>

            <table class="results">
                <thead>
                    <tr>
                        <th>Date</th>
                        <th>Open ({{ assumed }})</th>
                        <th>High ({{ assumed }})</th>
                        <th>Low ({{ assumed }})</th>
                        <th>Close ({{ assumed }})</th>
                        <th>Volume ({{ assumed }})</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(row, idx) in timeSeries.slice(0, 50)" :key="row.date">
                        <td>{{ row.date }}</td>
                        <td>{{ row.open }}</td>
                        <td>{{ row.high }}</td>
                        <td>{{ row.low }}</td>
                        <td>{{ row.close }}</td>
                        <td>{{ row.volume }}</td>
                    </tr>
                </tbody>
            </table>
            <h3>Converte Price</h3>
            <table class="results">
                <thead>
                    <tr>
                        <th>Date</th>
                        <th>Open ({{ converte }} Converte)</th>
                        <th>High ({{ converte }} Converte)</th>
                        <th>Low ({{ converte }} Converte)</th>
                        <th>Close ({{ converte }} Converte)</th>
                        <th>Volume ({{ converte }} Converte)</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="row in convertedTimeSeries.slice(0, 50)" :key="row.date">
                        <td>{{ row.date }}</td>
                        <td>{{ row.open }}</td>
                        <td>{{ row.high }}</td>
                        <td>{{ row.low }}</td>
                        <td>{{ row.close }}</td>
                        <td>{{ row.volume }}</td>
                    </tr>
                </tbody>
            </table>
        </template>

        <div v-else class="empty">No data yet.</div>
    </div>
</template>

<style scoped>
.av-table {
    font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
}

.controls {
    display: flex;
    gap: 8px;
    align-items: center;
    margin-bottom: 12px
}

.controls label {
    display: flex;
    flex-direction: column;
    font-size: 12px
}

.controls input {
    padding: 6px 8px;
    font-size: 14px
}

.controls button {
    padding: 6px 10px
}

.status {
    margin-bottom: 8px
}

.error {
    color: #c02626
}

.results {
    width: 100%;
    border-collapse: collapse
}

.results th,
.results td {
    border: 1px solid #ddd;
    padding: 6px 8px;
    text-align: right
}

.results th {
    background: #f5f5f5;
    text-align: left
}

.empty {
    color: #666
}
</style>