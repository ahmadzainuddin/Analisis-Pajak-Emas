<script setup>
import {
  CategoryScale,
  Chart as ChartJS,
  Filler,
  Legend,
  LineElement,
  LinearScale,
  PointElement,
  Tooltip,
} from 'chart.js'
import { computed, ref } from 'vue'
import { Line } from 'vue-chartjs'
import asbDividends from './data/asb-dividends.json'
import prices from './data/kijang-emas-prices.json'

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Filler, Tooltip, Legend)

const OUNCE_TO_GRAM = 31.1035

const weightGram = ref(20)
const pawnMarginPercent = ref(70)
const monthlyStoragePercent = ref(0.75)
const restructureMonths = ref(6)
const fdAnnualPercent = ref(3.5)
const isPriceModalOpen = ref(false)
const selectedInvestment = ref(null)

const sortedPrices = [...prices].sort((a, b) => new Date(a.date) - new Date(b.date))
const sortedAsbDividends = [...asbDividends].sort((a, b) => a.year - b.year)
const firstPrice = sortedPrices[0]
const latestPrice = sortedPrices.at(-1)
const firstMonth = firstPrice.date.slice(0, 7)
const latestMonth = latestPrice.date.slice(0, 7)
const startMonth = ref(firstMonth)
const endMonth = ref(latestMonth)

const currencyFormatter = new Intl.NumberFormat('ms-MY', {
  style: 'currency',
  currency: 'MYR',
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
})

const numberFormatter = new Intl.NumberFormat('ms-MY', {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
})

const dateFormatter = new Intl.DateTimeFormat('ms-MY', {
  month: 'short',
  year: 'numeric',
})

const fullDateFormatter = new Intl.DateTimeFormat('ms-MY', {
  day: '2-digit',
  month: 'short',
  year: 'numeric',
})

const formatCurrency = (value) =>
  currencyFormatter.format(Number.isFinite(value) ? value : 0).replace(/^RM\s*/u, 'RM ')
const formatNumber = (value) => numberFormatter.format(Number.isFinite(value) ? value : 0)
const formatDate = (date) => dateFormatter.format(new Date(date))
const formatFullDate = (date) => fullDateFormatter.format(new Date(date))
const formatYear = (date) => new Date(date).getFullYear()
const formatPercent = (value) => `${formatNumber(value)}%`

const scrollToRestructureTable = () => {
  document.getElementById('detailed-restructure-table')?.scrollIntoView({
    behavior: 'smooth',
    block: 'start',
  })
}

const monthStartDate = (monthValue) => new Date(`${monthValue || firstMonth}-01T00:00:00`)

const monthEndDate = (monthValue) => {
  const [year, month] = (monthValue || latestMonth).split('-').map(Number)
  return new Date(year, month, 0, 23, 59, 59)
}

const addMonths = (date, months) => {
  const next = new Date(date)
  next.setMonth(next.getMonth() + months)
  return next
}

const findPriceOnOrAfter = (targetDate, fallback = latestPrice) => {
  return sortedPrices.find((price) => new Date(price.date) >= targetDate) || fallback
}

const findPriceOnOrBefore = (targetDate, fallback = firstPrice) => {
  return [...sortedPrices].reverse().find((price) => new Date(price.date) <= targetDate) || fallback
}

const sellingPricePerGram = (price) => {
  // Harga jual/g ialah harga bank jual emas kepada pembeli.
  return price.one_oz_selling / OUNCE_TO_GRAM
}

const buyingPricePerGram = (price) => {
  // Harga beli/g ialah harga bank beli semula emas; ini digunakan untuk nilai pajak, restructure, dan jual secara konservatif.
  return price.one_oz_buying / OUNCE_TO_GRAM
}

const goldValue = (price) => {
  // Nilai emas untuk pajak/restructure/jual = harga beli/g * berat gram yang dimasukkan pengguna.
  return buyingPricePerGram(price) * Number(weightGram.value || 0)
}

const selectedRange = computed(() => {
  const startDate = monthStartDate(startMonth.value)
  const endDate = monthEndDate(endMonth.value)
  const rangeStartPrice = findPriceOnOrAfter(startDate, firstPrice)
  const rangeEndPrice = findPriceOnOrBefore(endDate, latestPrice)

  if (new Date(rangeStartPrice.date) > new Date(rangeEndPrice.date)) {
    return {
      startPrice: rangeEndPrice,
      endPrice: rangeEndPrice,
    }
  }

  return {
    startPrice: rangeStartPrice,
    endPrice: rangeEndPrice,
  }
})

const filteredPriceRows = computed(() => {
  const startDate = monthStartDate(startMonth.value)
  const endDate = monthEndDate(endMonth.value)

  if (startDate > endDate) {
    return sortedPrices.filter((price) => price.date === selectedRange.value.endPrice.date)
  }

  return sortedPrices.filter((price) => {
    const priceDate = new Date(price.date)
    return priceDate >= startDate && priceDate <= endDate
  })
})

const simulation = computed(() => {
  const margin = Number(pawnMarginPercent.value || 0) / 100
  const monthlyStorageRate = Number(monthlyStoragePercent.value || 0) / 100
  const intervalMonths = Math.max(1, Number(restructureMonths.value || 1))
  const startPrice = selectedRange.value.startPrice
  const endPrice = selectedRange.value.endPrice

  const initialGoldValue = goldValue(startPrice)
  const currentGoldValue = goldValue(endPrice)

  // Hasil gadaian awal = nilai emas awal * margin pajakan.
  const initialPawnProceeds = initialGoldValue * margin
  let pawnBalance = initialPawnProceeds
  let cumulativeRestructureCashflow = 0
  const rows = []
  let targetDate = addMonths(new Date(startPrice.date), intervalMonths)

  while (targetDate <= new Date(endPrice.date)) {
    const marketPrice = findPriceOnOrAfter(targetDate, endPrice)
    if (new Date(marketPrice.date) > new Date(endPrice.date)) break
    const currentPawnValue = goldValue(marketPrice) * margin

    // Hasil restructure ialah kenaikan pajakan baru berbanding baki pajakan lama.
    const restructureProceeds = currentPawnValue - pawnBalance
    // Upah simpan dikira atas baki pajakan lama untuk tempoh restructure.
    const storageFee = pawnBalance * monthlyStorageRate * intervalMonths
    const netSurplus = restructureProceeds - storageFee
    const positiveCashflow = Math.max(netSurplus, 0)

    cumulativeRestructureCashflow += positiveCashflow
    pawnBalance = currentPawnValue

    rows.push({
      date: marketPrice.date,
      buyingPricePerGram: buyingPricePerGram(marketPrice),
      goldValue: goldValue(marketPrice),
      pawnBalance,
      restructureProceeds,
      storageFee,
      netSurplus,
    })

    targetDate = addMonths(targetDate, intervalMonths)
  }

  // Kos tebus semasa = baki pajakan terakhir selepas simulasi restructure.
  const redemptionCost = pawnBalance
  // Baki bersih jual = nilai emas semasa - kos tebus semasa.
  const netAfterRedeemAndSell = currentGoldValue - redemptionCost
  // Total lebihan bersih jadual = lebihan positif ditolak lebihan negatif.
  const totalNetSurplus = rows.reduce((total, row) => total + row.netSurplus, 0)
  // Jumlah cashflow = gadaian awal + baki jual bersih + total lebihan bersih net.
  const totalCashflow = initialPawnProceeds + netAfterRedeemAndSell + totalNetSurplus

  return {
    initialGoldValue,
    initialPawnProceeds,
    currentGoldValue,
    redemptionCost,
    netAfterRedeemAndSell,
    cumulativeRestructureCashflow,
    totalNetSurplus,
    totalCashflow,
    rows,
  }
})

const summaryCards = computed(() => [
  {
    label: 'Berat emas',
    value: `${formatNumber(Number(weightGram.value || 0))}g`,
    detail: `Harga beli/g ${formatDate(selectedRange.value.startPrice.date)}: ${formatCurrency(buyingPricePerGram(selectedRange.value.startPrice))}`,
  },
  { label: `Nilai emas awal ${formatDate(selectedRange.value.startPrice.date)}`, value: formatCurrency(simulation.value.initialGoldValue), detail: formatDate(selectedRange.value.startPrice.date) },
  { label: 'Hasil gadaian awal', value: formatCurrency(simulation.value.initialPawnProceeds), detail: `${pawnMarginPercent.value}% margin pajakan` },
  { label: 'Nilai emas tamat', value: formatCurrency(simulation.value.currentGoldValue), detail: formatDate(selectedRange.value.endPrice.date) },
  { label: 'Kos tebus semasa', value: formatCurrency(simulation.value.redemptionCost), detail: 'Baki pajakan terakhir' },
  { label: 'Baki bersih jika tebus dan jual', value: formatCurrency(simulation.value.netAfterRedeemAndSell), detail: 'Nilai semasa - kos tebus' },
  { label: 'Total lebihan bersih', value: formatCurrency(simulation.value.totalNetSurplus), detail: 'Positif ditolak negatif' },
  { label: 'Jumlah keseluruhan cashflow', value: formatCurrency(simulation.value.totalCashflow), detail: 'Gadaian + baki jual + lebihan bersih' },
])

const comparisonRows = computed(() => [
  [`Hasil gadaian ${formatDate(selectedRange.value.startPrice.date)}`, simulation.value.initialPawnProceeds, 'Tunai awal diterima'],
  ['Kos tebus semasa', simulation.value.redemptionCost, 'Baki pajakan terakhir'],
  ['Nilai jual emas tamat', simulation.value.currentGoldValue, `Berdasarkan harga belian ${formatDate(selectedRange.value.endPrice.date)}`],
  ['Baki selepas tebus dan jual', simulation.value.netAfterRedeemAndSell, 'Nilai jual ditolak kos tebus'],
  ['Jumlah keseluruhan cash flow', simulation.value.totalCashflow, 'Gadaian awal + baki jual bersih + total lebihan bersih'],
])

const compoundValue = (principal, annualRate, years) => principal * (1 + annualRate) ** years

const asbDividendForYear = (year) => {
  return (
    sortedAsbDividends.find((item) => item.year === year) ||
    sortedAsbDividends.findLast((item) => item.year < year) ||
    sortedAsbDividends.at(-1)
  )
}

const compoundProfitRows = (principal, annualRate, startYear, years, yearlyRates = null) => {
  const rows = []
  let openingBalance = principal

  for (let index = 1; index <= years; index += 1) {
    const year = startYear + index - 1
    const rateInfo = yearlyRates ? yearlyRates(year) : null
    const rate = rateInfo ? rateInfo.total / 100 : annualRate
    const profit = openingBalance * rate
    const closingBalance = openingBalance + profit

    rows.push({
      year,
      rate,
      rateLabel: rateInfo
        ? `${formatNumber(rateInfo.dividend)} + ${formatNumber(rateInfo.bonus)}`
        : formatPercent(rate * 100),
      totalRatePercent: rate * 100,
      openingBalance,
      profit,
      closingBalance,
    })

    openingBalance = closingBalance
  }

  return rows
}

const compoundFromRows = (rows, principal) => rows.at(-1)?.closingBalance || principal

const investmentComparison = computed(() => {
  const principal = simulation.value.initialGoldValue
  const startYear = formatYear(selectedRange.value.startPrice.date)
  const endYear = formatYear(selectedRange.value.endPrice.date)
  const years = Math.max(0, endYear - startYear)
  const asbRows = compoundProfitRows(principal, null, startYear, years, asbDividendForYear)
  const asbValue = compoundFromRows(asbRows, principal)
  const fdRate = Number(fdAnnualPercent.value || 0) / 100
  const fdValue = compoundValue(principal, fdRate, years)

  return {
    years,
    rows: [
      {
        investment: 'Pajak emas + restructure',
        initial: principal,
        final: simulation.value.totalCashflow,
        totalProfit: simulation.value.totalCashflow - principal,
        returnPercent: ((simulation.value.totalCashflow / principal) - 1) * 100,
      },
      {
        investment: 'ASB',
        key: 'asb',
        rate: null,
        yearlyRates: asbDividendForYear,
        rateDescription: 'Ikut dividen + bonus tahunan',
        initial: principal,
        final: asbValue,
        totalProfit: asbValue - principal,
        returnPercent: ((asbValue / principal) - 1) * 100,
      },
      {
        investment: `FD (${formatNumber(Number(fdAnnualPercent.value || 0))}%)`,
        key: 'fd',
        rate: fdRate,
        rateDescription: `${formatPercent(Number(fdAnnualPercent.value || 0))} setahun`,
        initial: principal,
        final: fdValue,
        totalProfit: fdValue - principal,
        returnPercent: ((fdValue / principal) - 1) * 100,
      },
    ],
  }
})

const selectedInvestmentDetails = computed(() => {
  if (!selectedInvestment.value) return null

  const detail = investmentComparison.value.rows.find((row) => row.key === selectedInvestment.value)
  if (!detail) return null

  const startYear = formatYear(selectedRange.value.startPrice.date)
  const rows = compoundProfitRows(
    detail.initial,
    detail.rate,
    startYear,
    investmentComparison.value.years,
    detail.yearlyRates,
  )

  return {
    ...detail,
    showTotalRateColumn: detail.key === 'asb',
    rows,
    totalProfit: rows.reduce((total, row) => total + row.profit, 0),
    totalProfitPercent: ((detail.final / detail.initial) - 1) * 100,
  }
})

const chartData = computed(() => ({
  labels: simulation.value.rows.map((row) => formatDate(row.date)),
  datasets: [
    {
      label: 'Nilai emas',
      data: simulation.value.rows.map((row) => row.goldValue),
      borderColor: '#0f766e',
      backgroundColor: 'rgba(15, 118, 110, 0.12)',
      tension: 0.35,
      fill: true,
    },
    {
      label: 'Baki pajakan',
      data: simulation.value.rows.map((row) => row.pawnBalance),
      borderColor: '#1d4ed8',
      backgroundColor: 'rgba(29, 78, 216, 0.08)',
      tension: 0.35,
    },
    {
      label: 'Upah simpan',
      data: simulation.value.rows.map((row) => row.storageFee),
      borderColor: '#b45309',
      backgroundColor: 'rgba(180, 83, 9, 0.08)',
      tension: 0.35,
    },
    {
      label: 'Lebihan bersih',
      data: simulation.value.rows.map((row) => row.netSurplus),
      borderColor: '#7c3aed',
      backgroundColor: 'rgba(124, 58, 237, 0.08)',
      tension: 0.35,
    },
  ],
}))

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  interaction: {
    mode: 'index',
    intersect: false,
  },
  plugins: {
    legend: {
      position: 'bottom',
      labels: {
        boxWidth: 10,
        boxHeight: 10,
        usePointStyle: true,
      },
    },
    tooltip: {
      callbacks: {
        label: (context) => `${context.dataset.label}: ${formatCurrency(context.parsed.y)}`,
      },
    },
  },
  scales: {
    y: {
      ticks: {
        callback: (value) => formatCurrency(value),
      },
    },
  },
}
</script>

<template>
  <main class="dashboard-shell">
    <section class="hero">
      <div>
        <p class="eyebrow">Dashboard simulasi pajakan emas 999</p>
        <h1>Analisis Pajak Emas</h1>
        <p class="hero-copy">
          Penilaian konservatif berdasarkan harga belian Kijang Emas 999 1oz dari
          {{ formatDate(selectedRange.startPrice.date) }} hingga
          {{ formatDate(selectedRange.endPrice.date) }}. Dashboard ini membandingkan hasil strategi
          pajak emas dan restructure dengan alternatif simpanan ASB serta Fixed Deposit (FD), supaya
          perbezaan cashflow, nilai akhir, dan anggaran pulangan dapat dilihat dalam satu tempat.<br>
          https://www.bnm.gov.my/kijang-emas-prices
        </p>
      </div>
      <div class="market-snapshot">
        <span>Harga beli tamat/g</span>
        <strong>{{ formatCurrency(buyingPricePerGram(selectedRange.endPrice)) }}</strong>
      </div>
    </section>

    <section class="controls-panel" aria-label="Input simulasi">
      <label>
        <span>Berat emas dalam gram</span>
        <input v-model.number="weightGram" type="number" min="0" step="0.01" />
      </label>
      <label>
        <span>Margin pajakan (%)</span>
        <input v-model.number="pawnMarginPercent" type="number" min="0" max="100" step="0.1" />
      </label>
      <label>
        <span>Upah simpan bulanan (%)</span>
        <input v-model.number="monthlyStoragePercent" type="number" min="0" step="0.01" />
      </label>
      <label>
        <span>Tempoh restructure (bulan)</span>
        <input v-model.number="restructureMonths" type="number" min="1" step="1" />
      </label>
      <label>
        <span>FD tahunan (%)</span>
        <input v-model.number="fdAnnualPercent" type="number" min="0" step="0.01" />
      </label>
      <label>
        <span>Tarikh mula</span>
        <input v-model="startMonth" type="month" :min="firstMonth" :max="latestMonth" />
      </label>
      <label>
        <span>Tarikh tamat</span>
        <input v-model="endMonth" type="month" :min="firstMonth" :max="latestMonth" />
      </label>
      <div class="control-action">
        <button type="button" class="primary-button" @click="isPriceModalOpen = true">
          Harga Kijang Emas 999
        </button>
        <span>{{ filteredPriceRows.length }} rekod dalam julat tarikh</span>
      </div>
    </section>

    <section class="summary-grid" aria-label="Ringkasan utama">
      <article v-for="card in summaryCards" :key="card.label" class="summary-card">
        <span>{{ card.label }}</span>
        <strong>{{ card.value }}</strong>
        <small>{{ card.detail }}</small>
      </article>
    </section>

    <section class="content-grid">
      <article class="panel report-panel">
        <p class="section-label">Laporan</p>
        <h2>
          Analisis Strategi Pajak Emas 999 {{ formatNumber(weightGram) }}g
          ({{ formatYear(selectedRange.startPrice.date) }}-{{ formatYear(selectedRange.endPrice.date) }})
        </h2>
        <p>
          Hasil analisis menunjukkan strategi pajak semula emas
          {{ formatNumber(weightGram) }} gram dari tahun
          {{ formatYear(selectedRange.startPrice.date) }} hingga
          {{ formatYear(selectedRange.endPrice.date) }} masih memberikan pulangan positif.
        </p>
        <p>
          Daripada hasil gadaian awal sekitar
          {{ formatCurrency(simulation.initialPawnProceeds) }}, pemilik emas masih mampu menebus
          emas dan menjualnya semula dengan baki bersih sekitar
          {{ formatCurrency(simulation.netAfterRedeemAndSell) }}.
        </p>
        <p>
          Total lebihan bersih daripada proses restructure pula adalah sekitar
          {{ formatCurrency(simulation.totalNetSurplus) }}, selepas lebihan positif ditolak dengan
          lebihan negatif.
        </p>
        <p>
          Secara keseluruhan, jumlah cash flow yang pernah diterima sepanjang tempoh simulasi adalah
          sekitar {{ formatCurrency(simulation.totalCashflow) }}.
        </p>
        <p>
          Ini menunjukkan kenaikan harga emas berjaya menampung kos upah simpan dan masih memberikan
          lebihan nilai kepada pemilik emas.
        </p>
      </article>

      <article class="panel chart-panel">
        <div class="panel-heading">
          <div>
            <p class="section-label">Graf</p>
            <h2>Pergerakan nilai dan pajakan</h2>
          </div>
        </div>
        <div class="chart-wrap">
          <Line :data="chartData" :options="chartOptions" />
        </div>
      </article>
    </section>

    <section class="panel">
      <div class="panel-heading">
        <div>
          <p class="section-label">Jadual 1</p>
          <h2>Summary Comparison</h2>
        </div>
      </div>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Perkara</th>
              <th>Jumlah RM</th>
              <th>Catatan</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in comparisonRows" :key="row[0]">
              <td>{{ row[0] }}</td>
              <td>{{ formatCurrency(row[1]) }}</td>
              <td>{{ row[2] }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <section id="detailed-restructure-table" class="panel">
      <div class="panel-heading">
        <div>
          <p class="section-label">Jadual 2</p>
          <h2>Detailed Restructure Table</h2>
        </div>
      </div>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>Tarikh</th>
              <th>Harga beli emas/g</th>
              <th>Nilai emas</th>
              <th>Baki pajakan</th>
              <th>Hasil restructure</th>
              <th>Upah {{ restructureMonths }} bulan</th>
              <th>Lebihan bersih</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in simulation.rows" :key="row.date">
              <td>{{ formatDate(row.date) }}</td>
              <td>{{ formatCurrency(row.buyingPricePerGram) }}</td>
              <td>{{ formatCurrency(row.goldValue) }}</td>
              <td>{{ formatCurrency(row.pawnBalance) }}</td>
              <td>{{ formatCurrency(row.restructureProceeds) }}</td>
              <td>{{ formatCurrency(row.storageFee) }}</td>
              <td :class="{ positive: row.netSurplus >= 0, negative: row.netSurplus < 0 }">
                {{ formatCurrency(row.netSurplus) }}
              </td>
            </tr>
          </tbody>
          <tfoot>
            <tr>
              <td colspan="6">Total lebihan bersih</td>
              <td
                :class="{
                  positive: simulation.totalNetSurplus >= 0,
                  negative: simulation.totalNetSurplus < 0,
                }"
              >
                {{ formatCurrency(simulation.totalNetSurplus) }}
              </td>
            </tr>
          </tfoot>
        </table>
      </div>
    </section>

    <div v-if="isPriceModalOpen" class="modal-backdrop" @click.self="isPriceModalOpen = false">
      <section class="price-modal" role="dialog" aria-modal="true" aria-labelledby="price-modal-title">
        <div class="modal-heading">
          <div>
            <p class="section-label">Data JSON</p>
            <h2 id="price-modal-title">Harga Kijang Emas 999</h2>
            <p>
              {{ formatDate(selectedRange.startPrice.date) }} hingga
              {{ formatDate(selectedRange.endPrice.date) }} · {{ filteredPriceRows.length }} rekod
            </p>
          </div>
          <button type="button" class="icon-button" aria-label="Tutup" @click="isPriceModalOpen = false">
            x
          </button>
        </div>

        <div class="table-wrap modal-table-wrap">
          <table class="price-table">
            <thead>
              <tr>
                <th>Tarikh</th>
                <th>1oz Selling</th>
                <th>1oz Buying</th>
                <th>Harga jual/g</th>
                <th>Harga beli/g</th>
                <th>1/2oz Selling</th>
                <th>1/2oz Buying</th>
                <th>1/4oz Selling</th>
                <th>1/4oz Buying</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="price in filteredPriceRows" :key="price.date">
                <td>{{ formatFullDate(price.date) }}</td>
                <td>{{ formatCurrency(price.one_oz_selling) }}</td>
                <td>{{ formatCurrency(price.one_oz_buying) }}</td>
                <td>{{ formatCurrency(sellingPricePerGram(price)) }}</td>
                <td>{{ formatCurrency(buyingPricePerGram(price)) }}</td>
                <td>{{ formatCurrency(price.half_oz_selling) }}</td>
                <td>{{ formatCurrency(price.half_oz_buying) }}</td>
                <td>{{ formatCurrency(price.quarter_oz_selling) }}</td>
                <td>{{ formatCurrency(price.quarter_oz_buying) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </div>

    <section class="panel investment-panel">
      <div class="panel-heading">
        <div>
          <p class="section-label">Perbandingan Pelaburan</p>
          <h2>Comparison Table</h2>
        </div>
        <span class="comparison-note">
          Auto compounding {{ investmentComparison.years }} tahun
        </span>
      </div>
      <div class="table-wrap">
        <table class="comparison-table">
          <thead>
            <tr>
              <th>Pelaburan</th>
              <th>Modal Awal</th>
              <th>Nilai {{ formatYear(selectedRange.endPrice.date) }}</th>
              <th>Total Profit RM</th>
              <th>Anggaran Return</th>
              <th>Details</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in investmentComparison.rows" :key="row.investment">
              <td>
                <span class="highlight-investment">{{ row.investment }}</span>
              </td>
              <td>{{ formatCurrency(row.initial) }}</td>
              <td>{{ formatCurrency(row.final) }}</td>
              <td>{{ formatCurrency(row.totalProfit) }}</td>
              <td>{{ formatPercent(row.returnPercent) }}</td>
              <td>
                <button
                  v-if="row.key"
                  type="button"
                  class="details-button"
                  @click="selectedInvestment = row.key"
                >
                  View details
                </button>
                <button
                  v-else
                  type="button"
                  class="details-button"
                  @click="scrollToRestructureTable"
                >
                  View details
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <div
      v-if="selectedInvestmentDetails"
      class="modal-backdrop"
      @click.self="selectedInvestment = null"
    >
      <section class="price-modal" role="dialog" aria-modal="true" aria-labelledby="profit-modal-title">
        <div class="modal-heading">
          <div>
            <p class="section-label">Profit Compounding</p>
            <h2 id="profit-modal-title">{{ selectedInvestmentDetails.investment }}</h2>
            <p>
              Modal awal {{ formatCurrency(selectedInvestmentDetails.initial) }} ·
              {{ selectedInvestmentDetails.rateDescription }} ·
              {{ investmentComparison.years }} tahun
            </p>
          </div>
          <button type="button" class="icon-button" aria-label="Tutup" @click="selectedInvestment = null">
            x
          </button>
        </div>

        <div class="table-wrap modal-table-wrap">
          <table>
            <thead>
              <tr>
                <th>Tahun</th>
                <th>Kadar</th>
                <th v-if="selectedInvestmentDetails.showTotalRateColumn">Total %</th>
                <th>Baki awal</th>
                <th>Profit tahunan</th>
                <th>Baki selepas compounding</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in selectedInvestmentDetails.rows" :key="row.year">
                <td>{{ row.year }}</td>
                <td>{{ row.rateLabel }}</td>
                <td v-if="selectedInvestmentDetails.showTotalRateColumn">
                  {{ formatPercent(row.totalRatePercent) }}
                </td>
                <td>{{ formatCurrency(row.openingBalance) }}</td>
                <td>{{ formatCurrency(row.profit) }}</td>
                <td>{{ formatCurrency(row.closingBalance) }}</td>
              </tr>
            </tbody>
            <tfoot>
              <tr>
                <td :colspan="selectedInvestmentDetails.showTotalRateColumn ? 4 : 3">
                  Total profit
                </td>
                <td>{{ formatCurrency(selectedInvestmentDetails.totalProfit) }}</td>
                <td>{{ formatCurrency(selectedInvestmentDetails.final) }}</td>
              </tr>
              <tr>
                <td :colspan="selectedInvestmentDetails.showTotalRateColumn ? 4 : 3">
                  Total % profit
                </td>
                <td>{{ formatPercent(selectedInvestmentDetails.totalProfitPercent) }}</td>
                <td></td>
              </tr>
            </tfoot>
          </table>
        </div>
      </section>
    </div>

    <section class="author-panel" aria-labelledby="author-title">
      <div>
        <p class="section-label">Author</p>
        <h2 id="author-title">Ahmad Zainuddin</h2>
        <p>
          Financial dashboard, data analytics project, and Malaysian fintech simulation
          built with Vue.js and historical BNM Kijang Emas data.
        </p>
      </div>
      <div class="author-links" aria-label="Author and project links">
        <a href="https://github.com/ahmadzainuddin" target="_blank" rel="noopener noreferrer">
          <span>GitHub</span>
          <strong>@ahmadzainuddin</strong>
        </a>
        <a
          href="https://github.com/ahmadzainuddin/Analisis-Pajak-Emas"
          target="_blank"
          rel="noopener noreferrer"
        >
          <span>Project</span>
          <strong>Analisis Pajak Emas</strong>
        </a>
        <a href="https://analisis-pajak-emas.pages.dev/" target="_blank" rel="noopener noreferrer">
          <span>Live Demo</span>
          <strong>analisis-pajak-emas.pages.dev</strong>
        </a>
      </div>
    </section>
  </main>
</template>

<style>
:root {
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  color: #172033;
  background: #f4f6f8;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
}

button,
input {
  font: inherit;
}

button {
  cursor: pointer;
}

.dashboard-shell {
  width: min(1440px, 100%);
  margin: 0 auto;
  padding: 28px;
}

.hero {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  gap: 24px;
  align-items: end;
  padding: 28px 0 22px;
}

.eyebrow,
.section-label {
  margin: 0 0 8px;
  color: #607086;
  font-size: 0.78rem;
  font-weight: 700;
  letter-spacing: 0;
  text-transform: uppercase;
}

h1,
h2 {
  margin: 0;
  color: #121826;
  letter-spacing: 0;
}

h1 {
  font-size: clamp(2.2rem, 4vw, 4.4rem);
  line-height: 0.95;
}

h2 {
  font-size: 1.05rem;
}

.hero-copy {
  max-width: 760px;
  margin: 16px 0 0;
  color: #526174;
  font-size: 1.02rem;
  line-height: 1.65;
}

.market-snapshot,
.summary-card,
.panel,
.controls-panel {
  border: 1px solid #dfe5ec;
  border-radius: 8px;
  background: #ffffff;
  box-shadow: 0 12px 30px rgba(23, 32, 51, 0.06);
}

.market-snapshot {
  min-width: 240px;
  padding: 18px;
}

.market-snapshot span,
.summary-card span,
label span {
  display: block;
  color: #607086;
  font-size: 0.82rem;
  font-weight: 700;
}

.market-snapshot strong {
  display: block;
  margin-top: 8px;
  color: #0f766e;
  font-size: 1.6rem;
}

.controls-panel {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 14px;
  padding: 18px;
}

label {
  display: grid;
  gap: 8px;
}

.control-action {
  display: grid;
  gap: 8px;
  align-content: end;
}

.control-action span {
  color: #607086;
  font-size: 0.82rem;
}

.primary-button,
.icon-button {
  border: 0;
  border-radius: 8px;
  font-weight: 800;
}

.primary-button {
  min-height: 44px;
  padding: 0 16px;
  color: #ffffff;
  background: #0f766e;
}

.primary-button:hover {
  background: #115e59;
}

.icon-button {
  width: 38px;
  height: 38px;
  color: #526174;
  background: #eef2f6;
}

.icon-button:hover {
  color: #121826;
  background: #e2e8f0;
}

input {
  width: 100%;
  min-height: 44px;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 0 12px;
  color: #172033;
  background: #f8fafc;
}

input:focus {
  border-color: #0f766e;
  outline: 3px solid rgba(15, 118, 110, 0.16);
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 14px;
  margin-top: 18px;
}

.summary-card {
  min-height: 144px;
  min-width: 280px;
  padding: 18px;
}

.summary-card strong {
  display: block;
  margin-top: 14px;
  color: #121826;
  font-size: 1.35rem;
  line-height: 1.1;
  overflow-wrap: normal;
}

.summary-card small {
  display: block;
  margin-top: 12px;
  color: #6b7788;
  line-height: 1.35;
}

.content-grid {
  display: grid;
  grid-template-columns: minmax(320px, 0.85fr) minmax(0, 1.15fr);
  gap: 18px;
  margin-top: 18px;
}

.panel {
  padding: 22px;
  margin-top: 18px;
}

.content-grid .panel {
  margin-top: 0;
}

.report-panel p:not(.section-label) {
  color: #3f4b5d;
  line-height: 1.75;
}

.panel-heading {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  align-items: center;
  margin-bottom: 16px;
}

.comparison-note {
  color: #607086;
  font-size: 0.9rem;
  font-weight: 700;
}

.chart-wrap {
  height: 420px;
}

.table-wrap {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  min-width: 820px;
}

th,
td {
  border-bottom: 1px solid #e6ebf1;
  padding: 14px 12px;
  text-align: left;
  vertical-align: top;
}

th {
  color: #526174;
  font-size: 0.78rem;
  text-transform: uppercase;
}

td {
  color: #263246;
}

tfoot td {
  border-bottom: 0;
  border-top: 2px solid #cbd5e1;
  background: #f8fafc;
  font-weight: 800;
}

.positive {
  color: #047857;
  font-weight: 700;
}

.negative {
  color: #b91c1c;
  font-weight: 700;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 20;
  display: grid;
  place-items: center;
  padding: 24px;
  background: rgba(15, 23, 42, 0.56);
}

.price-modal {
  width: min(1180px, 100%);
  max-height: min(760px, 90vh);
  display: grid;
  grid-template-rows: auto minmax(0, 1fr);
  border: 1px solid #dfe5ec;
  border-radius: 8px;
  background: #ffffff;
  box-shadow: 0 24px 70px rgba(15, 23, 42, 0.26);
  overflow: hidden;
}

.modal-heading {
  display: flex;
  justify-content: space-between;
  gap: 18px;
  align-items: flex-start;
  padding: 22px;
  border-bottom: 1px solid #e6ebf1;
}

.modal-heading p:not(.section-label) {
  margin: 8px 0 0;
  color: #607086;
}

.modal-table-wrap {
  overflow: auto;
}

.price-table {
  min-width: 1240px;
}

.price-table th {
  position: sticky;
  top: 0;
  z-index: 1;
  background: #ffffff;
}

.highlight-investment {
  color: #0f766e;
  font-weight: 800;
}

.details-button {
  min-height: 36px;
  border: 1px solid #0f766e;
  border-radius: 8px;
  padding: 0 12px;
  color: #0f766e;
  background: #ffffff;
  font-weight: 800;
  white-space: nowrap;
}

.details-button:hover {
  color: #ffffff;
  background: #0f766e;
}

.muted-cell {
  color: #94a3b8;
}

.author-panel {
  display: grid;
  grid-template-columns: minmax(0, 0.9fr) minmax(360px, 1.1fr);
  gap: 18px;
  align-items: center;
  margin-top: 18px;
  padding: 22px;
  border: 1px solid #dfe5ec;
  border-radius: 8px;
  background: #ffffff;
  box-shadow: 0 12px 30px rgba(23, 32, 51, 0.06);
}

.author-panel p:not(.section-label) {
  max-width: 620px;
  margin: 10px 0 0;
  color: #526174;
  line-height: 1.6;
}

.author-links {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
}

.author-links a {
  display: grid;
  gap: 8px;
  min-height: 92px;
  align-content: center;
  border: 1px solid #dfe5ec;
  border-radius: 8px;
  padding: 14px;
  color: #172033;
  background: #f8fafc;
  text-decoration: none;
}

.author-links a:hover {
  border-color: #0f766e;
  background: #f0fdfa;
}

.author-links span {
  color: #607086;
  font-size: 0.78rem;
  font-weight: 800;
  text-transform: uppercase;
}

.author-links strong {
  color: #0f766e;
  font-size: 0.95rem;
  overflow-wrap: anywhere;
}

@media (max-width: 1180px) {
  .controls-panel {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .author-panel {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 860px) {
  .dashboard-shell {
    padding: 18px;
  }

  .hero,
  .content-grid {
    grid-template-columns: 1fr;
  }

  .market-snapshot {
    min-width: 0;
  }

  .controls-panel {
    grid-template-columns: 1fr;
  }

  .summary-card {
    min-height: 116px;
  }

  .chart-wrap {
    height: 340px;
  }

  .modal-backdrop {
    padding: 12px;
  }

  .modal-heading {
    padding: 18px;
  }

  .author-links {
    grid-template-columns: 1fr;
  }
}
</style>
