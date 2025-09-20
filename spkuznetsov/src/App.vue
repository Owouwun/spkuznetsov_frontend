<template>
  <div id="app">
    <header class="topbar">
      <h1>Orders Management — Мини UI</h1>
      <div class="actions">
        <button @click="fetchOrders" :disabled="loading">Загрузить заказы</button>
        <button @click="showCreate = !showCreate">{{ showCreate ? 'Скрыть' : 'Создать заказ' }}</button>
      </div>
    </header>

    <main class="main">
      <section v-if="showCreate" class="create">
        <h2>Новый заказ</h2>
        <form @submit.prevent="createOrder">
          <label>
            Имя клиента
            <input v-model="newOrder.client_name" required />
          </label>
          <label>
            Телефон
            <input v-model="newOrder.client_phone" />
          </label>
          <label>
            Адрес
            <input v-model="newOrder.address" />
          </label>
          <label>
            Описание
            <textarea v-model="newOrder.client_description"></textarea>
          </label>
          <div class="form-actions">
            <button type="submit" :disabled="creating">Создать</button>
            <button type="button" @click="resetNew">Очистить</button>
          </div>
        </form>
      </section>

      <section class="filter">
        <label>
          Поиск (имя или телефон)
          <input v-model="filter" placeholder="часть имени или телефона" />
        </label>
        <label>
          Статус
          <select v-model="filterStatus">
            <option value="">Все</option>
            <option v-for="(label, val) in statusMap" :key="val" :value="val">{{ label }}</option>
          </select>
        </label>
      </section>

      <section class="grid">
        <article v-for="order in filteredOrders" :key="order.id" class="card">
          <div class="card-header">
            <div class="title">
              <strong>{{ order.client_name || '—' }}</strong>
              <small>{{ order.client_phone || '' }}</small>
            </div>
            <div class="status" :data-status="order.status">{{ statusMap[order.status] ?? order.status }}</div>
          </div>

          <div class="card-body">
            <p class="addr">{{ order.address || 'Адрес не указан' }}</p>
            <p v-if="order.client_description"><em>{{ order.client_description }}</em></p>
            <p v-if="order.employee"><strong>Сотрудник:</strong> {{ order.employee.name }} (id: {{ order.employee.id }})</p>
            <p v-if="order.scheduled_for"><strong>Запланировано:</strong> {{ order.scheduled_for }}</p>
            <p v-if="order.cancel_reason"><strong>Причина отмены:</strong> {{ order.cancel_reason }}</p>
          </div>

          <div class="card-actions">
            <button @click="getOrder(order.id)">Обновить</button>
            <button @click="togglePanel(order.id, 'assign')">Назначить сотрудника</button>
            <button @click="togglePanel(order.id, 'preschedule')">Запланировать дату</button>
            <button @click="togglePanel(order.id, 'schedule')">Назначить дату</button>
            <button @click="togglePanel(order.id, 'progress')">Выполнены работы</button>
            <button @click="togglePanel(order.id, 'cancel')">Отменить</button>
            <button @click="patchSimple(order.id, '/close')">Закрыть</button>
            <button @click="patchSimple(order.id, '/complete')">Завершить</button>
          </div>

          <!-- inline panels -->
          <div v-if="panels[order.id]?.assign" class="panel">
            <label>Employee ID (integer)
              <input v-model.number="panels[order.id].assignVal" type="number" />
            </label>
            <div class="panel-actions">
              <button @click="assign(order.id, panels[order.id].assignVal)">Отправить</button>
              <button @click="togglePanel(order.id, 'assign')">Отмена</button>
            </div>
          </div>

          <div v-if="panels[order.id]?.cancel" class="panel">
            <label>Причина отмены
              <input v-model="panels[order.id].cancelVal" />
            </label>
            <div class="panel-actions">
              <button @click="cancel(order.id, panels[order.id].cancelVal)">Отправить</button>
              <button @click="togglePanel(order.id, 'cancel')">Отмена</button>
            </div>
          </div>

          <div v-if="panels[order.id]?.preschedule" class="panel">
            <label>Дата/время (ISO 8601)
              <input v-model="panels[order.id].prescheduleVal" placeholder="2025-09-20T14:00:00Z" />
            </label>
            <div class="panel-actions">
              <button @click="preschedule(order.id, panels[order.id].prescheduleVal)">Отправить</button>
              <button @click="togglePanel(order.id, 'preschedule')">Отмена</button>
            </div>
          </div>

          <div v-if="panels[order.id]?.schedule" class="panel">
            <label>Дата/время (ISO 8601)
              <input v-model="panels[order.id].scheduleVal" placeholder="2025-09-21T09:00:00Z" />
            </label>
            <div class="panel-actions">
              <button @click="schedule(order.id, panels[order.id].scheduleVal)">Отправить</button>
              <button @click="togglePanel(order.id, 'schedule')">Отмена</button>
            </div>
          </div>

          <div v-if="panels[order.id]?.progress" class="panel">
            <label>Описание работ (employee_description)
              <input v-model="panels[order.id].progressVal" />
            </label>
            <div class="panel-actions">
              <button @click="progress(order.id, panels[order.id].progressVal)">Отправить</button>
              <button @click="togglePanel(order.id, 'progress')">Отмена</button>
            </div>
          </div>
        </article>
      </section>

      <section v-if="error" class="error">
        <strong>Ошибка:</strong> {{ error }}
      </section>

      <footer class="footer">
        <small>API base: {{ baseUrl }}</small>
      </footer>
    </main>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'

// Базовый URL — можно переопределить через VITE_API_URL
const baseUrl = import.meta.env.VITE_API_URL || 'http://localhost:8080/api/v1'

const orders = ref([])
const loading = ref(false)
const creating = ref(false)
const error = ref(null)
const showCreate = ref(false)
const filter = ref('')
const filterStatus = ref('')

const panels = reactive({}) // panels[orderId] = { assign: true, assignVal: 5, ... }

// status mapping (взято из документации)
const statusMap = {
  0: 'Оформлена',
  1: 'Назначена предварительная дата работ',
  2: 'Назначен сотрудник',
  3: 'Назначены работы',
  4: 'Работы частично проведены',
  5: 'Выполнена',
  6: 'Оплачена',
  '-1': 'Отменена'
}

// Форма создания
const newOrder = reactive({
  client_name: '',
  client_phone: '',
  address: '',
  client_description: ''
})

function resetNew() {
  newOrder.client_name = ''
  newOrder.client_phone = ''
  newOrder.address = ''
  newOrder.client_description = ''
}

// Загрузить список заказов
async function fetchOrders() {
  loading.value = true
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders`)
    if (!res.ok) throw new Error(`${res.status} ${res.statusText}`)
    orders.value = await res.json()
  } catch (err) {
    error.value = err.message
  } finally {
    loading.value = false
  }
}

// Получить один заказ (обновить карточку)
async function getOrder(id) {
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders/${encodeURIComponent(id)}`)
    if (!res.ok) throw new Error(`${res.status} ${res.statusText}`)
    const updated = await res.json()
    const idx = orders.value.findIndex(o => o.id === id)
    if (idx >= 0) orders.value[idx] = updated
    else orders.value.unshift(updated)
  } catch (err) {
    error.value = err.message
  }
}

// Создать заказ
async function createOrder() {
  creating.value = true
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        address: newOrder.address,
        client_description: newOrder.client_description,
        client_name: newOrder.client_name,
        client_phone: newOrder.client_phone
      })
    })
    if (res.status === 201) {
      const text = await res.text()
      // сервер возвращает UUID (string). Попробуем получить полный объект заказа.
      await getOrder(text.trim())
      resetNew()
      showCreate.value = false
    } else {
      const payload = await parsePossibleJSON(res)
      throw new Error(`${res.status} ${res.statusText} — ${JSON.stringify(payload)}`)
    }
  } catch (err) {
    error.value = err.message
  } finally {
    creating.value = false
  }
}

// Утилиты: парсер JSON если есть
async function parsePossibleJSON(res) {
  const t = await res.text()
  try { return JSON.parse(t) } catch { return t }
}

// Простые патчи без тела: /close, /complete
async function patchSimple(id, suffix) {
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders/${encodeURIComponent(id)}${suffix}`, {
      method: 'PATCH'
    })
    if (!res.ok) {
      const payload = await parsePossibleJSON(res)
      throw new Error(`${res.status} ${res.statusText} — ${JSON.stringify(payload)}`)
    }
    await getOrder(id)
  } catch (err) {
    error.value = err.message
  }
}

// Assign
async function assign(id, empID) {
  if (empID == null || empID === '') { error.value = 'employee id required'; return }
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders/${encodeURIComponent(id)}/assign/${encodeURIComponent(empID)}`, {
      method: 'PATCH'
    })
    if (!res.ok) {
      const payload = await parsePossibleJSON(res)
      throw new Error(`${res.status} ${res.statusText} — ${JSON.stringify(payload)}`)
    }
    togglePanel(id, 'assign')
    await getOrder(id)
  } catch (err) {
    error.value = err.message
  }
}

// Cancel
async function cancel(id, reason) {
  if (!reason) { error.value = 'reason required'; return }
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders/${encodeURIComponent(id)}/cancel`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ cancel_reason: reason })
    })
    if (!res.ok) {
      const payload = await parsePossibleJSON(res)
      throw new Error(`${res.status} ${res.statusText} — ${JSON.stringify(payload)}`)
    }
    togglePanel(id, 'cancel')
    await getOrder(id)
  } catch (err) {
    error.value = err.message
  }
}

// Preschedule & Schedule (body: { scheduled_for: string })
async function preschedule(id, dt) {
  await scheduleCommon(id, dt, 'preschedule')
}
async function schedule(id, dt) {
  await scheduleCommon(id, dt, 'schedule')
}
async function scheduleCommon(id, dt, which) {
  if (!dt) { error.value = 'datetime required (ISO 8601)'; return }
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders/${encodeURIComponent(id)}/${which}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ scheduled_for: dt })
    })
    if (!res.ok) {
      const payload = await parsePossibleJSON(res)
      throw new Error(`${res.status} ${res.statusText} — ${JSON.stringify(payload)}`)
    }
    togglePanel(id, which)
    await getOrder(id)
  } catch (err) {
    error.value = err.message
  }
}

// Progress
async function progress(id, text) {
  if (!text) { error.value = 'progress text required'; return }
  error.value = null
  try {
    const res = await fetch(`${baseUrl}/orders/${encodeURIComponent(id)}/progress`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ employee_description: text })
    })
    if (!res.ok) {
      const payload = await parsePossibleJSON(res)
      throw new Error(`${res.status} ${res.statusText} — ${JSON.stringify(payload)}`)
    }
    togglePanel(id, 'progress')
    await getOrder(id)
  } catch (err) {
    error.value = err.message
  }
}

// переключение инлайн-панелей
function togglePanel(id, name) {
  if (!panels[id]) panels[id] = {}
  // toggle boolean
  panels[id][name] = !panels[id][name]
  // init values
  if (panels[id][name]) {
    if (name === 'assign') panels[id].assignVal = panels[id].assignVal ?? 0
    if (name === 'cancel') panels[id].cancelVal = panels[id].cancelVal ?? ''
    if (name === 'preschedule') panels[id].prescheduleVal = panels[id].prescheduleVal ?? ''
    if (name === 'schedule') panels[id].scheduleVal = panels[id].scheduleVal ?? ''
    if (name === 'progress') panels[id].progressVal = panels[id].progressVal ?? ''
  }
}

// фильтрация
const filteredOrders = computed(() => {
  return orders.value.filter(o => {
    const textMatch = !filter.value || ((o.client_name||'').toLowerCase().includes(filter.value.toLowerCase()) || (o.client_phone||'').includes(filter.value))
    const statusMatch = !filterStatus.value || String(o.status) === String(filterStatus.value)
    return textMatch && statusMatch
  })
})

// автозагрузка при старте
onMounted(() => {
  fetchOrders()
})
</script>

<style scoped>
:root{
  --bg:#0f1720;
  --card:#0b1220;
  --muted:#9aa4b2;
  --accent:#66c2b6;
  --danger:#e06c75;
  --glass: rgba(255,255,255,0.03);
}
#app{
  font-family: Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
  color: #e6eef6;
  background: linear-gradient(180deg,#071122 0%, #031021 100%);
  min-height:100vh;
  padding:20px;
}
.topbar{
  display:flex;
  justify-content:space-between;
  align-items:center;
  gap:12px;
  margin-bottom:18px;
}
.topbar h1{margin:0;font-size:1.2rem}
.actions button{margin-left:8px;padding:8px 12px;border-radius:8px;background:var(--glass);border:1px solid rgba(255,255,255,0.04);color:var(--accent);cursor:pointer}
.main{max-width:1100px;margin:0 auto}
.create, .filter{background:var(--card);padding:12px;border-radius:10px;margin-bottom:14px;box-shadow:0 6px 18px rgba(2,6,23,0.6)}
.create form label, .filter label{display:block;margin-bottom:8px;font-size:0.9rem;color:var(--muted)}
.create input, .create textarea, .filter input, .filter select{width:100%;padding:8px;margin-top:4px;border-radius:6px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit}
.form-actions{display:flex;gap:8px; margin-top:8px}
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));gap:12px}
.card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(0,0,0,0.04));padding:12px;border-radius:10px;border:1px solid rgba(255,255,255,0.03)}
.card-header{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
.title small{display:block;color:var(--muted);font-size:0.85rem}
.status{font-size:0.8rem;padding:6px 10px;border-radius:999px;background:rgba(255,255,255,0.02)}
.addr{color:var(--muted);margin:6px 0}
.card-actions{display:flex;flex-wrap:wrap;gap:6px;margin-top:8px}
.card-actions button{padding:6px 8px;border-radius:8px;color: white;background:transparent;border:1px solid rgba(255,255,255,0.03);cursor:pointer}
.panel{margin-top:10px;padding:8px;border-radius:8px;background:rgba(255,255,255,0.01);border:1px dashed rgba(255,255,255,0.02)}
.panel-actions{display:flex;gap:8px;margin-top:8px}
.error{margin-top:12px;padding:12px;border-radius:8px;background:rgba(224,108,117,0.12);color:var(--danger)}
.footer{margin-top:18px;color:var(--muted);text-align:center}
[data-status="0"]{background:rgba(102,194,182,0.07)}
[data-status="5"]{background:rgba(102,194,182,0.06)}
[data-status="-1"]{background:rgba(255, 255, 255, 0.08)}
</style>
