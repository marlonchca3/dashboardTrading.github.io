<template>
  <div class="app" :data-theme="theme">
    <header class="topbar">
      <div class="brand">
        <div class="logo">T</div>
        <div>
          <h1>Trading Dashboard</h1>
          <p class="muted">Watchlist · Posiciones · Órdenes · Riesgo (Vue)</p>
        </div>
      </div>

      <div class="actions">
        <div class="pill">
          <span class="dot" :class="{ ok: feedOk }"></span>
          <span>{{ feedOk ? "Feed OK" : "Feed OFF" }}</span>
        </div>

        <select v-model="account" class="select" aria-label="Account">
          <option value="SIM">SIM</option>
          <option value="LIVE">LIVE</option>
        </select>

        <button class="btn" @click="toggleTheme">
          {{ theme === "dark" ? "☀️ Claro" : "🌙 Oscuro" }}
        </button>
      </div>
    </header>

    <main class="grid">
      <!-- KPIs -->
      <section class="cards">
        <KpiCard title="Equity" :value="fmtMoney(equity)" sub="Cuenta" :trend="equityTrend" />
        <KpiCard title="P&L Hoy" :value="fmtMoney(pnlToday)" :sub="pnlToday >= 0 ? 'Verde' : 'Rojo'" :trend="pnlTrend" />
        <KpiCard
          title="Unrealized"
          :value="fmtMoney(unrealized)"
          sub="Posiciones abiertas"
          :trend="{ dir: unrealized >= 0 ? 'up' : 'down', pct: Math.round(Math.abs(unrealized) / 100 * 10) / 10 }"
        />
        <KpiCard
          title="DD Actual"
          :value="fmtMoney(drawdown)"
          sub="Desde peak"
          :trend="{ dir: drawdown <= 0 ? 'down' : 'flat', pct: Math.round(Math.abs(drawdown) / Math.max(1, peakEquity) * 1000) / 10 }"
        />
      </section>

      <!-- Watchlist -->
      <section class="panel">
        <div class="panel-head">
          <h2>Watchlist</h2>
          <div class="filters">
            <input v-model.trim="q" class="input" placeholder="Buscar símbolo…" aria-label="Buscar símbolo" />
            <button class="btn" @click="addSymbol">+ Agregar</button>
          </div>
        </div>

        <div class="table-wrap">
          <table class="table">
            <thead>
              <tr>
                <th>Símbolo</th>
                <th class="right">Last</th>
                <th class="right">Chg</th>
                <th class="right">Chg%</th>
                <th class="right">Spread</th>
                <th class="right">Vol</th>
                <th class="right">Acciones</th>
              </tr>
            </thead>

            <tbody>
              <tr
                v-for="r in filteredWatch"
                :key="r.symbol"
                @click="selectSymbol(r.symbol)"
                class="clickrow"
              >
                <td>
                  <span class="tag" :class="{ active: r.symbol === selected }">{{ r.symbol }}</span>
                </td>
                <td class="right">{{ r.last.toFixed(r.dp) }}</td>
                <td class="right" :class="r.chg >= 0 ? 'pos' : 'neg'">{{ fmtNum(r.chg, r.dp) }}</td>
                <td class="right" :class="r.chgPct >= 0 ? 'pos' : 'neg'">{{ r.chgPct.toFixed(2) }}%</td>
                <td class="right muted">{{ r.spread.toFixed(r.dp) }}</td>
                <td class="right muted">{{ fmtInt(r.vol) }}</td>
                <td class="right">
                  <button class="btn ghost" @click.stop="tradeMarket(r.symbol, 'BUY')">Buy</button>
                  <button class="btn ghost" @click.stop="tradeMarket(r.symbol, 'SELL')">Sell</button>
                </td>
              </tr>

              <tr v-if="filteredWatch.length === 0">
                <td colspan="7" class="empty">Sin símbolos.</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="panel-foot">
          <div class="muted small">Seleccionado: <b>{{ selected }}</b></div>
          <div class="muted small">Última actualización: {{ lastUpdate }}</div>
        </div>
      </section>

      <!-- Positions + Risk -->
      <section class="two">
        <section class="panel">
          <div class="panel-head">
            <h2>Posiciones</h2>
            <div class="filters">
              <button class="btn ghost" @click="flattenAll">Flatten</button>
            </div>
          </div>

          <div class="table-wrap">
            <table class="table">
              <thead>
                <tr>
                  <th>Símbolo</th>
                  <th class="right">Qty</th>
                  <th class="right">Avg</th>
                  <th class="right">Last</th>
                  <th class="right">Unrealized</th>
                  <th class="right">Stop</th>
                  <th class="right">Target</th>
                </tr>
              </thead>

              <tbody>
                <tr v-for="p in positions" :key="p.symbol">
                  <td><span class="tag">{{ p.symbol }}</span></td>
                  <td class="right" :class="p.qty > 0 ? 'pos' : 'neg'">{{ p.qty }}</td>
                  <td class="right">{{ p.avg.toFixed(p.dp) }}</td>
                  <td class="right">{{ lastOf(p.symbol).toFixed(p.dp) }}</td>
                  <td class="right" :class="p.unrealized >= 0 ? 'pos' : 'neg'">{{ fmtMoney(p.unrealized) }}</td>
                  <td class="right muted">{{ p.stop ? p.stop.toFixed(p.dp) : "—" }}</td>
                  <td class="right muted">{{ p.target ? p.target.toFixed(p.dp) : "—" }}</td>
                </tr>

                <tr v-if="positions.length === 0">
                  <td colspan="7" class="empty">Sin posiciones abiertas.</td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="panel-foot">
            <div class="muted">
              Unrealized total: <b :class="unrealized >= 0 ? 'pos' : 'neg'">{{ fmtMoney(unrealized) }}</b>
            </div>
            <button class="btn" @click="addRiskStops">Auto Stop/Target</button>
          </div>
        </section>

        <section class="panel">
          <div class="panel-head">
            <h2>Riesgo</h2>
            <div class="filters">
              <button class="btn ghost" @click="resetDay">Reset día</button>
            </div>
          </div>

          <div class="risk">
            <div class="risk-row">
              <div class="muted small">Max pérdida diaria</div>
              <div class="value">{{ fmtMoney(maxDailyLoss) }}</div>
            </div>

            <div class="risk-row">
              <div class="muted small">Pérdida actual del día</div>
              <div class="value" :class="dailyLoss <= 0 ? 'neg' : 'pos'">{{ fmtMoney(dailyLoss) }}</div>
            </div>

            <div class="risk-row">
              <div class="muted small">Riesgo por trade (estimado)</div>
              <div class="value">{{ fmtMoney(riskPerTrade) }}</div>
            </div>

            <div class="risk-row">
              <div class="muted small">Exposición (abs)</div>
              <div class="value">{{ fmtMoney(exposure) }}</div>
            </div>

            <div class="bar">
              <div class="bar-fill" :style="{ width: riskUsedPct + '%' }"></div>
            </div>

            <div class="muted small">
              Uso de riesgo: <b>{{ riskUsedPct.toFixed(1) }}%</b>
              <span v-if="riskUsedPct >= 100" class="neg"> · Límite alcanzado</span>
            </div>
          </div>
        </section>
      </section>

      <!-- Orders -->
      <section class="panel">
        <div class="panel-head">
          <h2>Órdenes</h2>
          <div class="filters">
            <select v-model="orderFilter" class="select" aria-label="Filtro órdenes">
              <option value="ALL">Todas</option>
              <option value="WORKING">Working</option>
              <option value="FILLED">Filled</option>
              <option value="CANCELLED">Cancelled</option>
            </select>
            <button class="btn danger ghost" @click="cancelWorking">Cancelar working</button>
          </div>
        </div>

        <div class="table-wrap">
          <table class="table">
            <thead>
              <tr>
                <th>Hora</th>
                <th>Símbolo</th>
                <th>Side</th>
                <th>Tipo</th>
                <th class="right">Qty</th>
                <th class="right">Precio</th>
                <th>Status</th>
              </tr>
            </thead>

            <tbody>
              <tr v-for="o in filteredOrders" :key="o.id">
                <td class="muted">{{ o.time }}</td>
                <td><span class="tag">{{ o.symbol }}</span></td>
                <td :class="o.side === 'BUY' ? 'pos' : 'neg'">{{ o.side }}</td>
                <td class="muted">{{ o.type }}</td>
                <td class="right">{{ o.qty }}</td>
                <td class="right">{{ o.price ? o.price.toFixed(o.dp) : "MKT" }}</td>
                <td><span class="badge" :class="o.status.toLowerCase()">{{ o.status }}</span></td>
              </tr>

              <tr v-if="filteredOrders.length === 0">
                <td colspan="7" class="empty">Sin órdenes.</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="panel-foot">
          <div class="muted">
            Realized hoy: <b :class="realized >= 0 ? 'pos' : 'neg'">{{ fmtMoney(realized) }}</b>
          </div>
          <button class="btn" @click="simulateFillOne">Simular Fill</button>
        </div>
      </section>
    </main>
  </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted, reactive, ref } from "vue";

/** UI */
const theme = ref("dark");
const account = ref("SIM");
const feedOk = ref(true);
const lastUpdate = ref(new Date().toLocaleTimeString());
const q = ref("");
const selected = ref("NQ");
const orderFilter = ref("ALL");

/** Demo state */
const state = reactive({
  equity: 10000,
  peakEquity: 10000,
  pnlToday: 120,
  realized: 80,
  maxDailyLoss: -250,
  watch: [
    { symbol: "NQ", dp: 2, last: 18000.25, prev: 17980.0, vol: 120000, spread: 0.25 },
    { symbol: "ES", dp: 2, last: 5100.5, prev: 5097.75, vol: 95000, spread: 0.25 },
    { symbol: "YM", dp: 0, last: 38950, prev: 38910, vol: 42000, spread: 1 },
    { symbol: "GC", dp: 1, last: 2035.6, prev: 2032.2, vol: 26000, spread: 0.2 },
  ],
  positions: [
    { symbol: "NQ", dp: 2, qty: 1, avg: 17990.25, stop: null, target: null, unrealized: 0 },
  ],
  orders: [
    { id: 1, time: nowHM(), symbol: "NQ", dp: 2, side: "BUY", type: "MKT", qty: 1, price: null, status: "FILLED" },
    { id: 2, time: nowHM(), symbol: "ES", dp: 2, side: "SELL", type: "LMT", qty: 1, price: 5102.0, status: "WORKING" },
  ],
});

/** Watch + delta */
const watchWithDelta = computed(() =>
  state.watch.map((r) => {
    const chg = r.last - r.prev;
    const chgPct = r.prev === 0 ? 0 : (chg / r.prev) * 100;
    return { ...r, chg, chgPct };
  })
);

const filteredWatch = computed(() => {
  const query = q.value.toLowerCase();
  return watchWithDelta.value.filter((r) => !query || r.symbol.toLowerCase().includes(query));
});

/** Portfolio metrics */
const equity = computed(() => state.equity);
const peakEquity = computed(() => state.peakEquity);
const drawdown = computed(() => state.equity - state.peakEquity);

const pnlToday = computed(() => state.pnlToday);
const realized = computed(() => state.realized);

const positions = computed(() => state.positions);
const unrealized = computed(() => state.positions.reduce((a, p) => a + p.unrealized, 0));

const dailyLoss = computed(() => (pnlToday.value < 0 ? pnlToday.value : 0));

const exposure = computed(() =>
  state.positions.reduce((a, p) => a + Math.abs(p.qty) * lastOf(p.symbol), 0)
);

const riskPerTrade = computed(() =>
  state.positions.reduce((a, p) => {
    if (!p.stop) return a;
    return a + Math.abs(p.avg - p.stop) * Math.abs(p.qty) * pointValue(p.symbol);
  }, 0)
);

const maxDailyLoss = computed(() => state.maxDailyLoss);

const riskUsedPct = computed(() => {
  const used = Math.abs(dailyLoss.value);
  const limit = Math.abs(maxDailyLoss.value);
  return limit === 0 ? 0 : (used / limit) * 100;
});

/** Trends */
const equityTrend = computed(() => {
  const dir = state.equity >= state.peakEquity ? "up" : "down";
  const pct =
    state.peakEquity === 0
      ? 0
      : ((state.equity - state.peakEquity) / Math.abs(state.peakEquity)) * 100;
  return { dir, pct: Math.round(pct * 10) / 10 };
});

const pnlTrend = computed(() => ({
  dir: state.pnlToday >= 0 ? "up" : "down",
  pct: Math.round((Math.abs(state.pnlToday) / 200) * 100),
}));

/** Orders */
const filteredOrders = computed(() => {
  if (orderFilter.value === "ALL") return state.orders;
  return state.orders.filter((o) => o.status === orderFilter.value);
});

/** Actions */
function toggleTheme() {
  theme.value = theme.value === "dark" ? "light" : "dark";
}

function selectSymbol(sym) {
  selected.value = sym;
}

function addSymbol() {
  const sym = prompt("Símbolo (ej: CL, NG, BTC, AAPL):");
  if (!sym) return;
  const s = sym.toUpperCase().trim();
  if (state.watch.some((x) => x.symbol === s)) return;
  state.watch.unshift({ symbol: s, dp: 2, last: 100, prev: 100, vol: 0, spread: 0.01 });
}

function tradeMarket(symbol, side) {
  const row = state.watch.find((w) => w.symbol === symbol);
  const dp = row?.dp ?? 2;
  const price = row?.last ?? 0;

  const order = {
    id: Date.now(),
    time: nowHM(),
    symbol,
    dp,
    side,
    type: "MKT",
    qty: 1,
    price: null,
    status: "FILLED",
  };
  state.orders.unshift(order);

  const sign = side === "BUY" ? 1 : -1;
  upsertPosition(symbol, sign, price, dp);
}

function upsertPosition(symbol, qtyDelta, fillPrice, dp) {
  const p = state.positions.find((x) => x.symbol === symbol);
  if (!p) {
    state.positions.push({
      symbol,
      dp,
      qty: qtyDelta,
      avg: fillPrice,
      stop: null,
      target: null,
      unrealized: 0,
    });
    return;
  }

  const newQty = p.qty + qtyDelta;

  if (p.qty !== 0 && Math.sign(p.qty) !== Math.sign(newQty) && newQty !== 0) {
    // flip
    state.realized += (fillPrice - p.avg) * p.qty * pointValue(symbol);
  } else if (newQty === 0) {
    // cierre total
    state.realized += (fillPrice - p.avg) * p.qty * pointValue(symbol);
    state.positions = state.positions.filter((x) => x.symbol !== symbol);
    return;
  } else if (Math.sign(p.qty) === Math.sign(newQty)) {
    // agrega misma dirección
    const oldCost = p.avg * p.qty;
    const addCost = fillPrice * qtyDelta;
    p.avg = (oldCost + addCost) / newQty;
  }

  p.qty = newQty;
  p.dp = dp;
}

function flattenAll() {
  const copy = [...state.positions];
  copy.forEach((p) => {
    const last = lastOf(p.symbol);
    const side = p.qty > 0 ? "SELL" : "BUY";
    state.orders.unshift({
      id: Date.now() + Math.random(),
      time: nowHM(),
      symbol: p.symbol,
      dp: p.dp,
      side,
      type: "MKT",
      qty: Math.abs(p.qty),
      price: null,
      status: "FILLED",
    });
    state.realized += (last - p.avg) * p.qty * pointValue(p.symbol);
  });
  state.positions = [];
}

function addRiskStops() {
  state.positions.forEach((p) => {
    const last = lastOf(p.symbol);
    const atr = last * 0.001; // demo
    const dir = Math.sign(p.qty);
    p.stop = last - dir * atr * 2;
    p.target = last + dir * atr * 3;
  });
}

function cancelWorking() {
  state.orders.forEach((o) => {
    if (o.status === "WORKING") o.status = "CANCELLED";
  });
}

function simulateFillOne() {
  const o = state.orders.find((x) => x.status === "WORKING");
  if (!o) return;
  o.status = "FILLED";
  const fillPrice = o.price ?? lastOf(o.symbol);
  const sign = o.side === "BUY" ? 1 : -1;
  upsertPosition(o.symbol, sign * o.qty, fillPrice, o.dp);
}

function resetDay() {
  state.pnlToday = 0;
  state.realized = 0;
  state.peakEquity = state.equity;
}

/** Helpers */
function nowHM() {
  return new Date().toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
}
function fmtMoney(n) {
  const sign = n < 0 ? "-" : "";
  const v = Math.abs(n);
  return `${sign}$${v.toLocaleString(undefined, { maximumFractionDigits: 0 })}`;
}
function fmtInt(n) {
  return Math.round(n).toLocaleString();
}
function fmtNum(n, dp = 2) {
  const sign = n < 0 ? "-" : "";
  const v = Math.abs(n);
  return `${sign}${v.toFixed(dp)}`;
}
function lastOf(symbol) {
  return state.watch.find((w) => w.symbol === symbol)?.last ?? 0;
}
function pointValue(symbol) {
  if (symbol === "NQ") return 20;
  if (symbol === "ES") return 50;
  if (symbol === "YM") return 5;
  if (symbol === "GC") return 100;
  return 1;
}

/** Simulated feed */
let timer = null;
onMounted(() => {
  timer = setInterval(() => {
    state.watch.forEach((r) => {
      r.prev = r.last;
      const step = (Math.random() - 0.5) * (r.symbol === "YM" ? 20 : r.symbol === "NQ" ? 8 : 2);
      r.last = Math.max(0, r.last + step);
      r.vol = Math.max(0, r.vol + Math.round(Math.random() * 2000));
    });

    state.positions.forEach((p) => {
      const last = lastOf(p.symbol);
      p.unrealized = (last - p.avg) * p.qty * pointValue(p.symbol);
    });

    state.pnlToday = state.realized + unrealized.value;
    state.equity = 10000 + state.pnlToday;
    if (state.equity > state.peakEquity) state.peakEquity = state.equity;

    if (Math.random() < 0.03) feedOk.value = false;
    if (Math.random() < 0.08) feedOk.value = true;

    lastUpdate.value = new Date().toLocaleTimeString();
  }, 1000);
});

onUnmounted(() => {
  if (timer) clearInterval(timer);
});

/** Local component */
const KpiCard = {
  props: ["title", "value", "sub", "trend"],
  template: `
    <div class="card">
      <div class="row">
        <div>
          <div class="muted small">{{ title }}</div>
          <div class="kpi">{{ value }}</div>
          <div class="muted small">{{ sub }}</div>
        </div>
        <div class="trend" :class="trend?.dir">
          <span v-if="trend?.dir === 'up'">▲</span>
          <span v-else-if="trend?.dir === 'down'">▼</span>
          <span v-else>●</span>
          <span class="small">{{ trend?.pct ?? 0 }}%</span>
        </div>
      </div>
    </div>
  `,
};
</script>

<style>
.app[data-theme="dark"]{
  --bg:#0b0f14; --card:#141821; --text:#eef2f7; --muted:#9aa4b2;
  --border:#2a3140; --btn:#1e2430; --btnHover:#2a3242; --accent:#5da9ff;
  --pos:#2bd576; --neg:#ff5b6a;
}
.app[data-theme="light"]{
  --bg:#f6f7fb; --card:#ffffff; --text:#0e1726; --muted:#57647a;
  --border:#e6e9f2; --btn:#f1f3f8; --btnHover:#e8ecf6; --accent:#2a6cff;
  --pos:#1fa85a; --neg:#d64252;
}
*{box-sizing:border-box}
body{margin:0}
.app{
  min-height:100vh;
  background:var(--bg);
  color:var(--text);
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
}

.topbar{
  display:flex; gap:16px; align-items:center; justify-content:space-between;
  padding:16px 18px; border-bottom:1px solid var(--border);
}
.brand{display:flex; gap:12px; align-items:center}
.logo{
  width:40px; height:40px; border-radius:14px;
  display:grid; place-items:center;
  background:var(--card); border:1px solid var(--border);
  font-weight:800;
}
h1{font-size:18px; margin:0}
.muted{color:var(--muted)}
.small{font-size:12px}

.actions{display:flex; gap:10px; align-items:center; flex-wrap:wrap}
.btn{
  background:var(--btn);
  border:1px solid var(--border);
  color:var(--text);
  padding:10px 12px;
  border-radius:12px;
  cursor:pointer;
}
.btn:hover{background:var(--btnHover)}
.btn.ghost{background:transparent}
.btn.danger{border-color:color-mix(in srgb, var(--neg), var(--border) 70%)}

.select,.input{
  background:var(--card);
  border:1px solid var(--border);
  color:var(--text);
  border-radius:12px;
  padding:10px 12px;
}

.pill{
  display:flex; gap:8px; align-items:center;
  padding:10px 12px;
  border-radius:999px;
  border:1px solid var(--border);
  background:var(--card);
}
.dot{width:10px; height:10px; border-radius:50%; background:var(--neg)}
.dot.ok{background:var(--pos)}

.grid{
  max-width:1200px;
  margin:0 auto;
  padding:18px;
  display:grid;
  gap:16px;
}
.cards{
  display:grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap:12px;
}
@media (max-width: 980px){
  .cards{grid-template-columns: repeat(2, minmax(0, 1fr));}
}
@media (max-width: 520px){
  .cards{grid-template-columns: 1fr;}
}

.card, .panel{
  background:var(--card);
  border:1px solid var(--border);
  border-radius:18px;
  padding:14px;
  box-shadow: 0 10px 25px rgba(0,0,0,.12);
}
.row{display:flex; justify-content:space-between; gap:12px; align-items:flex-start}
.kpi{font-size:26px; font-weight:800; margin:6px 0}

.trend{
  display:flex; gap:6px; align-items:center;
  padding:8px 10px; border-radius:12px;
  border:1px solid var(--border);
}
.trend.up{color:var(--pos)}
.trend.down{color:var(--neg)}
.trend.flat{color:var(--muted)}

.panel-head{
  display:flex;
  align-items:flex-start;
  justify-content:space-between;
  gap:12px;
  margin-bottom:12px;
}
.panel-head h2{margin:0; font-size:16px}
.filters{display:flex; gap:10px; flex-wrap:wrap; align-items:center}

.panel-foot{
  display:flex; justify-content:space-between; align-items:center;
  margin-top:12px; gap:12px; flex-wrap:wrap;
}

.two{
  display:grid;
  grid-template-columns: 2fr 1fr;
  gap:16px;
}
@media (max-width: 980px){
  .two{grid-template-columns: 1fr;}
}

.table-wrap{overflow:auto; border-radius:16px; border:1px solid var(--border)}
.table{width:100%; border-collapse:collapse; min-width:820px; background:transparent}
th, td{padding:12px; border-bottom:1px solid var(--border); text-align:left}
th{font-size:12px; color:var(--muted); font-weight:700}
.right{text-align:right}

.clickrow{cursor:pointer}
.clickrow:hover{background:color-mix(in srgb, var(--btnHover), transparent 60%)}

.tag{
  display:inline-block;
  padding:4px 10px;
  border:1px solid var(--border);
  border-radius:999px;
}
.tag.active{border-color: var(--accent); color: var(--accent)}

.badge{
  display:inline-block;
  padding:4px 10px;
  border-radius:999px;
  border:1px solid var(--border);
  font-size:12px;
}
.badge.working{color:var(--muted)}
.badge.filled{color:var(--pos)}
.badge.cancelled{color:var(--neg)}

.pos{color:var(--pos); font-weight:700}
.neg{color:var(--neg); font-weight:700}
.empty{padding:18px; text-align:center; color:var(--muted)}

.risk{display:grid; gap:12px}
.risk-row{
  display:flex; justify-content:space-between; gap:10px; align-items:baseline;
  border:1px solid var(--border);
  border-radius:14px;
  padding:12px;
}
.value{font-weight:800}
.bar{
  height:10px; border-radius:999px; overflow:hidden;
  border:1px solid var(--border);
  background:transparent;
}
.bar-fill{
  height:100%;
  background:var(--accent);
}
</style>