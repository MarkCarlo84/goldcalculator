<script setup>
import { ref, onMounted } from 'vue'

// --- Name (who's using the calculator) ---
const STORAGE_KEY = 'goldcalc_userName'

function getStoredName() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    return raw ? raw : null
  } catch {
    return null
  }
}

function setStoredName(name) {
  if (name) localStorage.setItem(STORAGE_KEY, name)
  else localStorage.removeItem(STORAGE_KEY)
}

const user = ref(null) // when set, { name: string } for display
const enterName = ref('')
const nameError = ref('')

function submitName() {
  nameError.value = ''
  const name = enterName.value.trim()
  if (!name) {
    nameError.value = 'Please enter your name.'
    return
  }
  setStoredName(name)
  user.value = { name }
  enterName.value = ''
}

function logout() {
  setStoredName(null)
  user.value = null
  nameError.value = ''
}

onMounted(() => {
  const name = getStoredName()
  if (name) user.value = { name }
})

// --- Calculator ---
const KARATS = [24, 22, 18, 14, 10]

// Price per gram (₱) by karat — editable (reference: GoldPriceZ Philippines)
const priceByKarat = ref({
  24: 9945.09,
  22: 9116.33,
  18: 7458.82,
  14: 5801.37,
  10: 4143.79,
})

// Separate calculator state per purity (no dropdown / no tabs)
const calcByKarat = ref(
  Object.fromEntries(
    KARATS.map((k) => [
      k,
      {
        weightGrams: '',
        designFeeType: 'fixed', // 'fixed' | 'percent'
        designFeeAmount: 500,
        designFeePercent: 10,
        history: [],
        historyOffset: 0,
      },
    ])
  )
)

const toFiniteNumber = (v) => {
  const n = typeof v === 'number' ? v : Number(v)
  return Number.isFinite(n) ? n : 0
}

const round2 = (n) => Math.round(toFiniteNumber(n) * 100) / 100

const pricePerGramFor = (k) => toFiniteNumber(priceByKarat.value[k])
const weightFor = (k) => toFiniteNumber(calcByKarat.value[k].weightGrams)

const goldValueFor = (k) => round2(weightFor(k) * pricePerGramFor(k))

const designFeeFor = (k) => {
  const c = calcByKarat.value[k]
  if (c.designFeeType === 'fixed') return toFiniteNumber(c.designFeeAmount)
  return round2((goldValueFor(k) * toFiniteNumber(c.designFeePercent) / 100))
}

const subtotalFor = (k) => round2(goldValueFor(k) + designFeeFor(k))
const taxFor = (k) => round2(subtotalFor(k) * 0.12)
const totalPriceFor = (k) => round2(subtotalFor(k) + taxFor(k))

const canCalculateFor = (k) => {
  const w = weightFor(k)
  const price24k = toFiniteNumber(priceByKarat.value[24])
  const priceK = pricePerGramFor(k)
  return w > 0 && price24k > 0 && priceK > 0
}

const formatPHP = (n) => {
  return '₱' + Number(n).toLocaleString('en-PH', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
}

const HISTORY_PAGE_SIZE = 1

const visibleHistoryFor = (k) =>
  calcByKarat.value[k].history.slice(
    calcByKarat.value[k].historyOffset,
    calcByKarat.value[k].historyOffset + HISTORY_PAGE_SIZE
  )

const canGoNextFor = (k) => calcByKarat.value[k].historyOffset > 0
const canGoPrevFor = (k) =>
  calcByKarat.value[k].historyOffset + HISTORY_PAGE_SIZE < calcByKarat.value[k].history.length

function goNext(k) {
  calcByKarat.value[k].historyOffset = Math.max(0, calcByKarat.value[k].historyOffset - HISTORY_PAGE_SIZE)
}

function goPrev(k) {
  calcByKarat.value[k].historyOffset = Math.min(
    calcByKarat.value[k].history.length - HISTORY_PAGE_SIZE,
    calcByKarat.value[k].historyOffset + HISTORY_PAGE_SIZE
  )
}

function addToHistory(k) {
  calcByKarat.value[k].history.unshift({
    id: Date.now(),
    weight: weightFor(k),
    karat: k,
    goldValue: goldValueFor(k),
    designFee: designFeeFor(k),
    tax: taxFor(k),
    total: totalPriceFor(k),
    at: new Date().toLocaleString('en-PH', { dateStyle: 'short', timeStyle: 'short' }),
  })
  calcByKarat.value[k].historyOffset = 0
  if (calcByKarat.value[k].history.length > 20) calcByKarat.value[k].history = calcByKarat.value[k].history.slice(0, 20)
}

function clearHistory(k) {
  calcByKarat.value[k].history = []
  calcByKarat.value[k].historyOffset = 0
}
</script>

<template>
  <div class="app-shell">
    <!-- Enter name (who's using the calculator) -->
    <div v-if="!user" class="auth-screen">
      <header class="auth-header">
        <h1 class="title">Gold Calculator</h1>
        <p class="subtitle">Enter your name to continue</p>
        <div class="header-accent"></div>
      </header>
      <div class="auth-card-wrapper">
        <div class="card auth-card">
          <h2 class="card-heading">
            <span class="card-heading-icon">◇</span>
            Enter Name
          </h2>
          <form class="auth-form" @submit.prevent="submitName">
            <div class="field">
              <label for="enter-name">Your name</label>
              <input
                id="enter-name"
                v-model="enterName"
                type="text"
                autocomplete="name"
                placeholder="e.g. Juan"
              />
            </div>
            <p v-if="nameError" class="auth-error">{{ nameError }}</p>
            <button type="submit" class="btn-auth">Continue</button>
          </form>
        </div>
      </div>
    </div>

    <!-- Calculator (when logged in) -->
    <div v-else class="calculator">
      <header class="header">
        <div class="header-top">
          <div class="user-bar">
            <span class="user-name">Hello, {{ user.name }}</span>
            <button type="button" class="btn-logout" @click="logout">Log out</button>
          </div>
        </div>
        <h1 class="title">Gold Calculator</h1>
        <p class="subtitle">Philippine Peso — live rate per gram by purity</p>
        <div class="header-accent"></div>
      </header>

      <nav class="purity-nav" aria-label="Purity navigation">
        <a v-for="k in KARATS" :key="k" class="purity-link" :href="`#karat-${k}`">{{ k }}K</a>
      </nav>

      <div class="karat-groups">
        <section v-for="k in KARATS" :key="k" class="karat-group" :id="`karat-${k}`">
          <h2 class="karat-group-title">{{ k }}K Calculator</h2>
          <div class="cards-row">
            <div class="card inputs-card">
              <h3 class="card-heading">
                <span class="card-heading-icon">◇</span>
                Calculator
              </h3>
              <div class="field">
                <label :for="`price-${k}`">{{ k }}K gold price per gram (₱)</label>
                <input
                  :id="`price-${k}`"
                  v-model.number="priceByKarat[k]"
                  type="number"
                  min="0.01"
                  step="0.01"
                  :placeholder="`e.g. ${Number(priceByKarat[k] || 0).toFixed(2)}`"
                />
                <span class="hint">
                  Get live rate from
                  <a href="https://goldpricez.com/ph/gram" target="_blank" rel="noopener">GoldPriceZ Philippines</a>
                  — update the <strong>{{ k }}K</strong> rate per gram here.
                </span>
              </div>

              <div class="field">
                <label :for="`weight-${k}`">Weight (grams)</label>
                <input
                  :id="`weight-${k}`"
                  v-model="calcByKarat[k].weightGrams"
                  type="number"
                  min="0.01"
                  step="0.1"
                  placeholder="e.g. 5"
                />
              </div>

              <div class="design-fee-section">
                <h4 class="section-title">Design fee</h4>
                <div class="design-fee-type">
                  <label class="radio-label">
                    <input v-model="calcByKarat[k].designFeeType" type="radio" value="fixed" />
                    Fixed amount (₱)
                  </label>
                  <label class="radio-label">
                    <input v-model="calcByKarat[k].designFeeType" type="radio" value="percent" />
                    Percentage of gold value
                  </label>
                </div>
                <div class="field design-fee-input">
                  <input
                    v-if="calcByKarat[k].designFeeType === 'fixed'"
                    v-model.number="calcByKarat[k].designFeeAmount"
                    type="number"
                    min="0"
                    step="50"
                    placeholder="e.g. 500"
                  />
                  <input
                    v-else
                    v-model.number="calcByKarat[k].designFeePercent"
                    type="number"
                    min="0"
                    max="100"
                    step="1"
                    placeholder="e.g. 10"
                  />
                  <span v-if="calcByKarat[k].designFeeType === 'percent'" class="suffix">%</span>
                </div>
              </div>

              <button
                type="button"
                class="btn-calculate"
                :disabled="!canCalculateFor(k)"
                @click="addToHistory(k)"
              >
                Calculate
              </button>
            </div>

            <div class="card results-card">
              <h3 class="card-heading">
                <span class="card-heading-icon">◆</span>
                Price details
              </h3>
              <div class="formula-note">
                <strong>Formula:</strong> Gold Value = Weight (g) × {{ k }}K price/g (matches
                <a href="https://goldpricez.com/ph/gram" target="_blank" rel="noopener">GoldPriceZ Philippines</a>)
              </div>
              <div class="result-row effective-rate">
                <span class="result-label">{{ k }}K per gram</span>
                <span class="result-value">{{ formatPHP(pricePerGramFor(k)) }}</span>
              </div>
              <div class="result-row gold-value">
                <span class="result-label">Gold value</span>
                <span class="result-value">{{ formatPHP(goldValueFor(k)) }}</span>
              </div>
              <div class="result-row design-fee-row">
                <span class="result-label">Design fee</span>
                <span class="result-value">{{ formatPHP(designFeeFor(k)) }}</span>
              </div>
              <div class="result-row tax-row">
                <span class="result-label">Tax (12%)</span>
                <span class="result-value">{{ formatPHP(taxFor(k)) }}</span>
              </div>
              <div class="result-row total">
                <span class="result-label">Total price</span>
                <span class="result-value total-value">{{ formatPHP(totalPriceFor(k)) }}</span>
              </div>

              <div class="history-section">
                <h4 class="history-title">
                  <span class="history-title-icon">📋</span>
                  Calculation history
                </h4>
                <div v-if="calcByKarat[k].history.length === 0" class="history-empty">
                  No calculations yet. Enter values and click Calculate.
                </div>
                <template v-else>
                  <ul class="history-list">
                    <li v-for="entry in visibleHistoryFor(k)" :key="entry.id" class="history-item">
                      <span class="history-summary">{{ entry.weight }}g {{ entry.karat }}K → {{ formatPHP(entry.total) }}</span>
                      <span class="history-meta">{{ entry.at }}</span>
                      <span class="history-detail">
                        {{ formatPHP(entry.goldValue) }} gold + {{ formatPHP(entry.designFee) }} fee + {{ formatPHP(entry.tax ?? 0) }} tax
                      </span>
                    </li>
                  </ul>
                  <div class="history-nav">
                    <button type="button" class="btn-history-nav" :disabled="!canGoPrevFor(k)" @click="goPrev(k)">
                      Previous
                    </button>
                    <span class="history-range">{{ calcByKarat[k].historyOffset + 1 }} of {{ calcByKarat[k].history.length }}</span>
                    <button type="button" class="btn-history-nav" :disabled="!canGoNextFor(k)" @click="goNext(k)">
                      Next
                    </button>
                  </div>
                  <button type="button" class="btn-clear-history" @click="clearHistory(k)">Clear history</button>
                </template>
              </div>
            </div>
          </div>
        </section>
      </div>

    <footer class="footer">
      <p>24K rate from <a href="https://goldpricez.com/ph/gram" target="_blank" rel="noopener">GoldPriceZ Philippines (gold per gram)</a>. Prices in Philippine Peso (₱).</p>
    </footer>
    </div>
  </div>
</template>

<style scoped>
.app-shell {
  width: 100%;
  max-width: 1100px;
  overflow: visible;
  display: flex;
  flex-direction: column;
}

/* Auth screen */
.auth-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  min-height: 100vh;
  max-height: 100%;
  overflow: auto;
  padding: 1.5rem 1rem;
  box-sizing: border-box;
}

.auth-header {
  text-align: center;
  margin-bottom: 1.5rem;
  flex-shrink: 0;
}

.auth-header .header-badge,
.auth-header .title,
.auth-header .subtitle,
.auth-header .header-accent {
  margin-left: auto;
  margin-right: auto;
}

.auth-card-wrapper {
  width: 100%;
  max-width: 400px;
  flex-shrink: 0;
  margin: 0 auto;
}

.auth-card {
  padding: 1.25rem 1.5rem;
}

.auth-form {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.auth-form .field {
  margin-top: 0;
}

.auth-error {
  font-size: 0.8rem;
  color: #f87171;
  margin: 0;
  padding: 0.35rem 0.5rem;
  background: rgba(248, 113, 113, 0.12);
  border-radius: 6px;
  border-left: 3px solid #f87171;
}

.btn-auth {
  width: 100%;
  margin-top: 0.25rem;
  padding: 0.65rem 1rem;
  font-size: 0.95rem;
  font-weight: 600;
  font-family: inherit;
  color: #1a1a1a;
  background: linear-gradient(145deg, #e8c735 0%, #d4af37 50%, #a67c0a 100%);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.15s ease, box-shadow 0.2s ease;
  box-shadow: 0 2px 8px rgba(184, 134, 11, 0.25);
}

.btn-auth:hover {
  box-shadow: 0 6px 20px rgba(212, 175, 55, 0.4);
  transform: translateY(-2px);
}

/* Calculator header with user */
.header-top {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 0.4rem;
}

.user-bar {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.user-name {
  font-size: 0.8rem;
  color: var(--text-muted);
}

.btn-logout {
  padding: 0.3rem 0.6rem;
  font-size: 0.75rem;
  font-weight: 500;
  font-family: inherit;
  color: var(--gold-muted);
  background: rgba(212, 175, 55, 0.12);
  border: 1px solid rgba(212, 175, 55, 0.3);
  border-radius: 6px;
  cursor: pointer;
  transition: border-color 0.2s, background 0.2s, color 0.2s;
}

.btn-logout:hover {
  border-color: var(--gold-soft);
  background: rgba(212, 175, 55, 0.18);
  color: var(--gold-bright);
}

.calculator {
  width: 100%;
  max-width: 1100px;
  max-height: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.cards-row {
  display: flex;
  gap: 1rem;
  align-items: stretch;
  flex-wrap: wrap;
  flex: 1;
  min-height: 0;
}

.cards-row .card {
  flex: 1;
  min-width: 320px;
  margin-bottom: 0;
  transition: box-shadow 0.25s ease, transform 0.2s ease;
}

.cards-row .card:hover {
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3), 0 0 28px rgba(212, 175, 55, 0.18);
}

@media (max-width: 720px) {
  .cards-row {
    flex-direction: column;
  }
  .cards-row .card {
    margin-bottom: 1rem;
  }
  .cards-row .card:last-child {
    margin-bottom: 0;
  }
}

.header {
  text-align: center;
  margin-bottom: 0.75rem;
  flex-shrink: 0;
}

.header-badge {
  display: inline-block;
  font-size: 0.65rem;
  font-weight: 600;
  letter-spacing: 0.08em;
  color: var(--gold-bright);
  background: rgba(212, 175, 55, 0.18);
  padding: 0.25rem 0.55rem;
  border-radius: 999px;
  margin-bottom: 0.4rem;
  border: 1px solid rgba(212, 175, 55, 0.35);
  box-shadow: 0 0 16px rgba(212, 175, 55, 0.15);
}

.title {
  font-family: 'Playfair Display', serif;
  font-size: 1.85rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  margin: 0 0 0.2rem 0;
  background: linear-gradient(135deg, #e8c735 0%, #d4af37 50%, #c9a227 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  filter: drop-shadow(0 0 12px rgba(212, 175, 55, 0.35));
}

.subtitle {
  font-size: 0.85rem;
  color: var(--text-muted);
  margin: 0 0 0.4rem 0;
}

.header-accent {
  width: 4rem;
  height: 3px;
  margin: 0 auto;
  background: linear-gradient(90deg, transparent, var(--gold-soft), var(--gold-bright), var(--gold-soft), transparent);
  border-radius: 2px;
  box-shadow: 0 0 10px rgba(212, 175, 55, 0.4);
}

.purity-nav {
  position: sticky;
  top: 0;
  z-index: 5;
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  flex-wrap: wrap;
  padding: 0.5rem 0.25rem;
  margin: 0 0 0.75rem 0;
  background: rgba(10, 10, 10, 0.78);
  backdrop-filter: blur(8px);
  border: 1px solid rgba(212, 175, 55, 0.18);
  border-radius: 12px;
}

.purity-link {
  text-decoration: none;
  padding: 0.35rem 0.65rem;
  font-size: 0.78rem;
  font-weight: 700;
  color: var(--text-muted);
  background: var(--surface-elevated);
  border: 1px solid var(--border);
  border-radius: 999px;
  transition: border-color 0.2s ease, color 0.2s ease, background 0.2s ease, box-shadow 0.2s ease;
}

.purity-link:hover {
  color: var(--text);
  border-color: rgba(212, 175, 55, 0.4);
  box-shadow: 0 0 14px rgba(212, 175, 55, 0.12);
}

.card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 1rem 1.15rem;
  margin-bottom: 0;
  box-shadow: var(--shadow-card);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.card-heading {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.95rem;
  font-weight: 600;
  color: var(--text);
  margin: 0 0 0.65rem 0;
  padding-bottom: 0.5rem;
  border-bottom: 1px solid var(--border);
  flex-shrink: 0;
}

.card-heading-icon {
  font-size: 0.75rem;
  color: var(--gold-soft);
}

.inputs-card .card-heading {
  margin-top: 0;
}

.results-card .card-heading {
  margin-top: 0;
}

.cards-row .card.inputs-card {
  flex: 0 1 420px;
}

.cards-row .card.results-card {
  flex: 1 1 380px;
  min-width: 340px;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
}

.karat-groups {
  display: flex;
  flex-direction: column;
  gap: 1.15rem;
  flex: 1;
  min-height: 0;
  overflow: visible;
  padding-right: 0.25rem;
}

.karat-group-title {
  margin: 0 0 0.5rem 0.1rem;
  font-size: 0.95rem;
  font-weight: 700;
  color: var(--gold-bright);
  text-shadow: 0 0 12px rgba(212, 175, 55, 0.25);
}

.btn-calculate {
  width: 100%;
  margin-top: 0.85rem;
  padding: 0.65rem 1rem;
  font-size: 0.95rem;
  font-weight: 600;
  font-family: inherit;
  color: #1a1a1a;
  background: linear-gradient(145deg, #e8c735 0%, #d4af37 50%, #a67c0a 100%);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.15s ease, box-shadow 0.2s ease;
  box-shadow: 0 2px 8px rgba(184, 134, 11, 0.25);
  flex-shrink: 0;
}

.btn-calculate:hover:not(:disabled) {
  box-shadow: 0 6px 20px rgba(212, 175, 55, 0.4);
  transform: translateY(-2px);
}

.btn-calculate:active:not(:disabled) {
  transform: translateY(0);
  box-shadow: 0 2px 8px rgba(184, 134, 11, 0.3);
}

.btn-calculate:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.inputs-card .field + .field,
.inputs-card .field + .design-fee-section {
  margin-top: 0.6rem;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
}

.field label {
  font-size: 0.78rem;
  font-weight: 500;
  color: var(--text-muted);
}

.field input,
.field select {
  width: 100%;
  padding: 0.45rem 0.65rem;
  font-size: 0.95rem;
  font-family: inherit;
  color: var(--text);
  background: var(--surface-elevated);
  border: 1px solid var(--border);
  border-radius: 8px;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.field input:focus,
.field select:focus {
  outline: none;
  border-color: var(--gold-soft);
  box-shadow: 0 0 0 3px rgba(212, 175, 55, 0.18);
}

.hint {
  font-size: 0.7rem;
  color: var(--text-muted);
}

.hint a {
  color: var(--gold-soft);
  text-decoration: none;
}

.hint a:hover {
  text-decoration: underline;
}

.design-fee-section {
  padding-top: 0.5rem;
  border-top: 1px solid var(--border);
}

.section-title {
  font-size: 0.82rem;
  font-weight: 600;
  margin: 0 0 0.35rem 0;
}

.design-fee-type {
  display: flex;
  flex-wrap: wrap;
  gap: 0.65rem;
  margin-bottom: 0.35rem;
}

.radio-label {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.85rem;
  cursor: pointer;
}

.radio-label input {
  width: auto;
  accent-color: var(--gold-soft);
}

.design-fee-input {
  position: relative;
}

.design-fee-input .suffix {
  position: absolute;
  right: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  color: var(--text-muted);
  font-size: 0.9rem;
}

.formula-note {
  font-size: 0.7rem;
  color: var(--text-muted);
  margin-bottom: 0.5rem;
  padding: 0.4rem 0.55rem;
  background: var(--surface-elevated);
  border-radius: 8px;
  border-left: 4px solid var(--gold-soft);
  box-shadow: 0 0 12px rgba(212, 175, 55, 0.08);
  flex-shrink: 0;
}

.formula-note strong {
  color: var(--gold-muted);
}

.formula-note a {
  color: var(--gold-soft);
  text-decoration: none;
}

.formula-note a:hover {
  text-decoration: underline;
}

.result-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.35rem 0;
  transition: background 0.15s ease;
  flex-shrink: 0;
}

.result-row + .result-row {
  border-top: 1px solid var(--border);
}

.result-row:hover {
  background: rgba(212, 175, 55, 0.04);
  margin: 0 -0.35rem;
  padding-left: 0.35rem;
  padding-right: 0.35rem;
  padding-top: 0.35rem;
  padding-bottom: 0.35rem;
  border-radius: 6px;
}

.result-label {
  font-size: 0.82rem;
  color: var(--text-muted);
}

.result-value {
  font-weight: 600;
  font-size: 0.95rem;
}

.result-row.total {
  margin-top: 0.35rem;
  margin-bottom: -0.15rem;
  padding: 0.55rem 0.5rem;
  background: linear-gradient(135deg, rgba(212, 175, 55, 0.18) 0%, rgba(184, 134, 11, 0.12) 100%);
  border-radius: 8px;
  border: 1px solid rgba(212, 175, 55, 0.4);
  box-shadow: 0 0 20px rgba(212, 175, 55, 0.12);
}

.result-row.total:hover {
  background: linear-gradient(135deg, rgba(212, 175, 55, 0.14) 0%, rgba(184, 134, 11, 0.1) 100%);
  margin: 0.35rem -0.15rem -0.15rem -0.15rem;
  padding: 0.55rem 0.5rem;
}

.result-row.total .result-value {
  font-size: 1.2rem;
  letter-spacing: -0.02em;
}

.total-value {
  color: var(--gold-bright);
  font-weight: 700;
  text-shadow: 0 0 12px rgba(212, 175, 55, 0.3);
}

.history-section {
  margin-top: 0.65rem;
  padding-top: 0.65rem;
  border-top: 1px solid var(--border);
  flex: 1;
  min-height: 0;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.history-title {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.82rem;
  font-weight: 600;
  margin: 0 0 0.4rem 0;
  color: var(--text-muted);
  flex-shrink: 0;
}

.history-title-icon {
  font-size: 0.85rem;
  opacity: 0.9;
}

.history-empty {
  font-size: 0.78rem;
  color: var(--text-muted);
  padding: 0.5rem 0.6rem;
  background: var(--surface-elevated);
  border-radius: 8px;
  text-align: center;
}

.history-list {
  list-style: none;
  margin: 0;
  padding: 0;
  flex: 1;
  min-height: 0;
  max-height: 8rem;
  overflow-y: auto;
}

.history-item {
  display: grid;
  grid-template-columns: 1fr auto;
  grid-template-rows: auto auto;
  gap: 0.15rem 0.65rem;
  padding: 0.4rem 0.45rem;
  font-size: 0.8rem;
  border-bottom: 1px solid var(--border);
  border-radius: 6px;
  transition: background 0.15s ease;
}

.history-item:hover {
  background: rgba(212, 175, 55, 0.06);
}

.history-item:last-child {
  border-bottom: none;
}

.history-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  margin-top: 0.4rem;
  padding-top: 0.35rem;
  flex-shrink: 0;
}

.history-range {
  font-size: 0.72rem;
  color: var(--text-muted);
}

.btn-history-nav {
  padding: 0.3rem 0.6rem;
  font-size: 0.75rem;
  font-weight: 500;
  font-family: inherit;
  color: var(--gold-muted);
  background: var(--surface-elevated);
  border: 1px solid var(--border);
  border-radius: 6px;
  cursor: pointer;
  transition: border-color 0.2s ease, background 0.2s ease, color 0.2s ease;
}

.btn-history-nav:hover:not(:disabled) {
  border-color: var(--gold-soft);
  background: rgba(212, 175, 55, 0.12);
  color: var(--gold-muted);
}

.btn-history-nav:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.history-summary {
  font-weight: 600;
  color: var(--text);
}

.history-meta {
  font-size: 0.75rem;
  color: var(--text-muted);
  text-align: right;
}

.history-detail {
  grid-column: 1 / -1;
  font-size: 0.8rem;
  color: var(--text-muted);
}

.btn-clear-history {
  margin-top: 0.4rem;
  padding: 0.35rem 0.6rem;
  font-size: 0.75rem;
  font-family: inherit;
  color: var(--text-muted);
  background: transparent;
  border: 1px solid var(--border);
  border-radius: 6px;
  cursor: pointer;
  transition: color 0.2s ease, border-color 0.2s ease, background 0.2s ease;
  flex-shrink: 0;
}

.btn-clear-history:hover {
  color: var(--text);
  border-color: var(--text-muted);
  background: var(--surface-elevated);
}

.footer {
  text-align: center;
  margin-top: 0.5rem;
  padding-top: 0.5rem;
  border-top: 1px solid var(--border);
  flex-shrink: 0;
}

.footer p {
  font-size: 0.7rem;
  color: var(--text-muted);
  margin: 0;
  max-width: 32rem;
  margin-left: auto;
  margin-right: auto;
  line-height: 1.4;
}

.footer a {
  color: var(--gold-soft);
  text-decoration: none;
  font-weight: 500;
}

.footer a:hover {
  text-decoration: underline;
}
</style>
