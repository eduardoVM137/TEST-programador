<script setup>
import { ref, computed } from 'vue'
import 'devextreme/dist/css/dx.light.css'
import DxDateBox from 'devextreme-vue/date-box'
import DateRangesPicker from './components/DateRangesPicker.vue'

const facturaBaseId = ref(null)
const referencia = ref('')
const comentarios = ref('')

const horasProgramacionDisponibles = ['08:00', '09:00', '10:00', '12:00', '15:00', '18:00']

const opcionesDiasSemana = [
  { text: 'Lunes', short: 'L', value: '0' },
  { text: 'Martes', short: 'M', value: '1' },
  { text: 'Miércoles', short: 'X', value: '2' },
  { text: 'Jueves', short: 'J', value: '3' },
  { text: 'Viernes', short: 'V', value: '4' },
  { text: 'Sábado', short: 'S', value: '5' },
  { text: 'Domingo', short: 'D', value: '6' }
]

const opcionesQuincena = [
  { text: 'Día 1 del mes', value: '1' },
  { text: 'Día 15 del mes', value: '15' }
]

function buildHoraDate(hora = '09:00') {
  const [hh, mm] = hora.split(':').map(Number)
  const d = new Date()
  d.setHours(hh || 9, mm || 0, 0, 0)
  return d
}

function crearIntervalo(index = 1) {
  return {
    id: `tmp_${Date.now()}_${index}`,
    nombre: `Inter ${index}`,
    tipoPeriodicidad: 'F',
    s01226instruccionGeneracion: '',
    reglaIntervaloDias: 1,
    reglaFechaInicioUi: null,
    horaProgramadaUi: buildHoraDate('09:00'),
    fechasConfig: {
      rangos: [],
      reglas: []
    }
  }
}

const intervalosFactura = ref([crearIntervalo(1)])
const intervaloActivoId = ref(intervalosFactura.value[0].id)

const intervaloActivo = computed(() =>
  intervalosFactura.value.find(x => x.id === intervaloActivoId.value) || null
)

const tipoPeriodicidad = computed({
  get: () => intervaloActivo.value?.tipoPeriodicidad || 'F',
  set: v => {
    if (!intervaloActivo.value) return
    intervaloActivo.value.tipoPeriodicidad = v

    if (v !== 'S' && v !== 'Q') {
      intervaloActivo.value.s01226instruccionGeneracion = ''
    }

    if (v !== 'F' && !intervaloActivo.value.fechasConfig) {
      intervaloActivo.value.fechasConfig = { rangos: [], reglas: [] }
    }
  }
})

const reglaFechaInicioUi = computed({
  get: () => intervaloActivo.value?.reglaFechaInicioUi || null,
  set: v => {
    if (intervaloActivo.value) intervaloActivo.value.reglaFechaInicioUi = v
  }
})

const horaProgramadaUi = computed({
  get: () => intervaloActivo.value?.horaProgramadaUi || null,
  set: v => {
    if (intervaloActivo.value) intervaloActivo.value.horaProgramadaUi = v
  }
})

const reglaIntervaloDias = computed({
  get: () => intervaloActivo.value?.reglaIntervaloDias || 1,
  set: v => {
    if (intervaloActivo.value) intervaloActivo.value.reglaIntervaloDias = Number(v || 1)
  }
})

const pillsIntervalosFactura = computed(() =>
  intervalosFactura.value.map((item, index) => ({
    id: item.id,
    text: item.nombre || `Inter ${index + 1}`,
    hint: getTextoPillIntervalo(item)
  }))
)

const horaSeleccionadaTexto = computed(() => formatoHoraMilitar(horaProgramadaUi.value))

const resumenProgramacionHumano = computed(() => {
  const item = intervaloActivo.value
  if (!item) return 'Selecciona o crea un intervalo.'

  const inicio = item.reglaFechaInicioUi
    ? ` Inicio: ${formatoFechaCorta(item.reglaFechaInicioUi)}.`
    : ' Inicio: inmediato (desde hoy).'

  const hora = item.horaProgramadaUi
    ? ` Hora: ${formatoHoraCorta(item.horaProgramadaUi)}.`
    : ' Hora: sin definir.'

  if (item.tipoPeriodicidad === 'S') {
    const dias = getTextoInstruccion('S', item.s01226instruccionGeneracion) || 'sin días seleccionados'
    return `Se generará semanalmente los ${dias}.${inicio}${hora}`
  }

  if (item.tipoPeriodicidad === 'Q') {
    const dias = getTextoInstruccion('Q', item.s01226instruccionGeneracion) || 'sin cortes seleccionados'
    return `Se generará quincenalmente en ${dias}.${inicio}${hora}`
  }

  if (item.tipoPeriodicidad === 'F') {
    const reglasFecha = getResumenFechas(item)
    return `Se generará con reglas por fecha: ${reglasFecha}.${inicio}${hora}`
  }

  return `Se generará cada ${item.reglaIntervaloDias || 1} día(s).${inicio}${hora}`
})

const payloadVistaPrevia = computed(() => {
  return {
    facturaBaseId: facturaBaseId.value,
    referencia: referencia.value,
    comentarios: comentarios.value,
    intervalos: intervalosFactura.value.map(item => ({
      id: item.id,
      nombre: item.nombre,
      tipoPeriodicidad: item.tipoPeriodicidad,
      fechaInicio: formatoFechaCorta(item.reglaFechaInicioUi),
      hora: formatoHoraMilitar(item.horaProgramadaUi),
      instruccion: item.s01226instruccionGeneracion || '',
      intervaloDias: Number(item.reglaIntervaloDias || 1),
      rangos: (item.fechasConfig?.rangos || []).map(r => ({
        inicio: r.inicio,
        fin: r.fin
      })),
      reglas: (item.fechasConfig?.reglas || []).map(r => ({ ...r }))
    }))
  }
})

function formatoFechaCorta(value) {
  if (!value) return ''
  const fecha = new Date(value)
  if (Number.isNaN(fecha.getTime())) return ''
  return `${fecha.getFullYear()}-${String(fecha.getMonth() + 1).padStart(2, '0')}-${String(fecha.getDate()).padStart(2, '0')}`
}

function formatoHoraCorta(value) {
  if (!value) return ''
  const fecha = new Date(value)
  if (Number.isNaN(fecha.getTime())) return ''
  return fecha.toLocaleTimeString('es-MX', { hour: 'numeric', minute: '2-digit' })
}

function formatoHoraMilitar(value) {
  if (!value) return ''
  const fecha = new Date(value)
  if (Number.isNaN(fecha.getTime())) return ''
  return `${String(fecha.getHours()).padStart(2, '0')}:${String(fecha.getMinutes()).padStart(2, '0')}`
}

function getTextoInstruccion(intervalo, instruccion) {
  const arr = String(instruccion || '').split(',').map(x => x.trim()).filter(Boolean)

  if (!arr.length) return ''

  if (intervalo === 'S') {
    const dias = {
      '0': 'lunes',
      '1': 'martes',
      '2': 'miércoles',
      '3': 'jueves',
      '4': 'viernes',
      '5': 'sábado',
      '6': 'domingo'
    }
    return arr.map(x => dias[x] || x).join(', ')
  }

  if (intervalo === 'Q') {
    const periodos = { '1': 'día 1 del mes', '15': 'día 15 del mes' }
    return arr.map(x => periodos[x] || `día ${x}`).join(', ')
  }

  return arr.join(', ')
}

function getTextoPillIntervalo(item) {
  if (item.tipoPeriodicidad === 'X') {
    return `Cada ${item.reglaIntervaloDias || 1} día(s)`
  }

  if (item.tipoPeriodicidad === 'F') {
    return getResumenFechas(item, true)
  }

  return getTextoInstruccion(item.tipoPeriodicidad, item.s01226instruccionGeneracion) || 'Sin regla'
}

function getResumenFechas(item, corto = false) {
  const rangos = (item.fechasConfig?.rangos || []).slice()
  const reglas = (item.fechasConfig?.reglas || []).slice()

  const partesRangos = rangos.map(r => getTextoRango(r))
  const partesReglas = reglas.map(r => getTextoRegla(r))

  const partes = [...partesRangos, ...partesReglas]

  if (!partes.length) {
    return corto ? 'Sin fechas' : 'sin fechas seleccionadas'
  }

  const visibles = partes.slice(0, corto ? 2 : partes.length).join(corto ? ' | ' : ' · ')

  if (corto && partes.length > 2) {
    return `${visibles} +${partes.length - 2}`
  }

  return visibles
}

function getTextoRango(rango) {
  if (!rango) return ''

  if (rango.fin == null || rango.fin === '') {
    return `desde ${rango.inicio}`
  }

  if (rango.inicio === rango.fin) {
    return rango.inicio
  }

  return `${rango.inicio} al ${rango.fin}`
}

function getTextoRegla(regla) {
  if (!regla) return ''

  if (regla.tipo === 'monthly_day') {
    return regla.label || `Día ${regla.day} de cada mes`
  }

  if (regla.tipo === 'monthly_last_day') {
    return regla.label || 'Último día de cada mes'
  }

  if (regla.tipo === 'monthly_business_days') {
    return regla.label || 'Días hábiles de cada mes'
  }

  if (regla.tipo === 'monthly_range') {
    return regla.label || `Días ${regla.dayStart}-${regla.dayEnd} de cada mes`
  }

  return 'Regla'
}

function agregarIntervaloFactura() {
  const index = intervalosFactura.value.length + 1
  const nuevo = crearIntervalo(index)
  intervalosFactura.value.push(nuevo)
  intervaloActivoId.value = nuevo.id
}

function seleccionarIntervaloFacturaById(id) {
  intervaloActivoId.value = id
}

function eliminarIntervaloFactura(id) {
  if (intervalosFactura.value.length <= 1) return

  const idx = intervalosFactura.value.findIndex(x => x.id === id)
  if (idx < 0) return

  intervalosFactura.value.splice(idx, 1)

  intervalosFactura.value.forEach((x, i) => {
    x.nombre = `Inter ${i + 1}`
  })

  intervaloActivoId.value = intervalosFactura.value[0].id
}

function setFrecuenciaVisual(value) {
  tipoPeriodicidad.value = value
}

function isSchedulerOptionSelected(value) {
  const current = String(intervaloActivo.value?.s01226instruccionGeneracion || '')
    .split(',')
    .map(x => x.trim())
    .filter(Boolean)

  return current.includes(String(value))
}

function toggleSchedulerOption(value) {
  let current = String(intervaloActivo.value?.s01226instruccionGeneracion || '')
    .split(',')
    .map(x => x.trim())
    .filter(Boolean)

  const target = String(value)

  if (current.includes(target)) {
    current = current.filter(x => x !== target)
  } else {
    current.push(target)
  }

  current = current.sort((a, b) => Number(a) - Number(b))
  intervaloActivo.value.s01226instruccionGeneracion = current.join(',')
}

function seleccionarQuincenaRapida(values) {
  intervaloActivo.value.s01226instruccionGeneracion = values.join(',')
}

function decrementarIntervaloDias() {
  reglaIntervaloDias.value = Math.max(1, Number(reglaIntervaloDias.value || 1) - 1)
}

function incrementarIntervaloDias() {
  reglaIntervaloDias.value = Math.min(365, Number(reglaIntervaloDias.value || 1) + 1)
}

function seleccionarHoraRapida(hora) {
  horaProgramadaUi.value = buildHoraDate(hora)
}
</script>

<template>
  <div class="page">
    <div class="modal">
      <div class="header">
        <h2>Configuración de Factura Programada</h2>
      </div>

      <div class="content">
        <div class="grid-top">
          <div class="card">
            <h3>Plantilla base</h3>

            <label>Factura base</label>
            <input v-model="facturaBaseId" type="number" placeholder="ID de factura existente" />

            <label>Referencia base</label>
            <input v-model="referencia" type="text" placeholder="Referencia base" />

            <label>Comentarios</label>
            <textarea v-model="comentarios" rows="4" placeholder="Comentarios" />
          </div>

          <div class="card">
            <div class="fp-builder-v3">
              <div class="fp-section">
                <div class="fp-label">Intervalos de esta programación</div>

                <div class="fp-pills-wrap">
                  <button
                    v-for="item in pillsIntervalosFactura"
                    :key="item.id"
                    type="button"
                    class="fp-mini-pill fp-mini-pill-editable"
                    :class="{ active: String(intervaloActivoId) === String(item.id) }"
                    @click="seleccionarIntervaloFacturaById(item.id)"
                  >
                    <span>{{ item.text }}</span>
                    <span class="fp-mini-pill-hint">{{ item.hint }}</span>
                    <span
                      v-if="intervalosFactura.length > 1"
                      class="fp-pill-remove"
                      @click.stop="eliminarIntervaloFactura(item.id)"
                    >×</span>
                  </button>
                </div>

                <div class="fp-add-row">
                  <button type="button" class="fp-outline-btn" @click="agregarIntervaloFactura">
                    + Agregar intervalo
                  </button>
                </div>
              </div>

              <div class="fp-section" v-if="intervaloActivo">
                <div class="fp-step-title">
                  <span class="fp-step-badge">1</span>
                  Selecciona la frecuencia
                </div>

                <div class="fp-frequency-grid">
                  <button
                    type="button"
                    class="fp-frequency-card"
                    :class="{ active: tipoPeriodicidad === 'S' }"
                    @click="setFrecuenciaVisual('S')"
                  >
                    <strong>Semanal</strong>
                    <span>Días de la semana</span>
                  </button>

                  <button
                    type="button"
                    class="fp-frequency-card"
                    :class="{ active: tipoPeriodicidad === 'Q' }"
                    @click="setFrecuenciaVisual('Q')"
                  >
                    <strong>Quincenal</strong>
                    <span>Día 1, 15 o ambos</span>
                  </button>

                  <button
                    type="button"
                    class="fp-frequency-card"
                    :class="{ active: tipoPeriodicidad === 'F' }"
                    @click="setFrecuenciaVisual('F')"
                  >
                    <strong>Fechas</strong>
                    <span>Días, rangos, reglas mensuales y calendario</span>
                  </button>

                  <button
                    type="button"
                    class="fp-frequency-card"
                    :class="{ active: tipoPeriodicidad === 'X' }"
                    @click="setFrecuenciaVisual('X')"
                  >
                    <strong>Personalizado</strong>
                    <span>Cada X días</span>
                  </button>
                </div>
              </div>

              <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'S'">
                <div class="fp-step-title">
                  <span class="fp-step-badge">2</span>
                  Selecciona los días de la semana
                </div>

                <div class="fp-chip-grid days">
                  <button
                    v-for="dia in opcionesDiasSemana"
                    :key="dia.value"
                    type="button"
                    class="fp-chip-card"
                    :class="{ active: isSchedulerOptionSelected(dia.value) }"
                    @click="toggleSchedulerOption(dia.value)"
                  >
                    {{ dia.short }}
                    <small>{{ dia.text }}</small>
                  </button>
                </div>
              </div>

              <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'Q'">
                <div class="fp-step-title">
                  <span class="fp-step-badge">2</span>
                  Selecciona los cortes
                </div>

                <div class="fp-quick-row">
                  <button type="button" class="fp-quick-chip" @click="seleccionarQuincenaRapida(['1'])">
                    Primer día
                  </button>
                  <button type="button" class="fp-quick-chip" @click="seleccionarQuincenaRapida(['15'])">
                    Día 15
                  </button>
                  <button type="button" class="fp-quick-chip" @click="seleccionarQuincenaRapida(['1', '15'])">
                    1 y 15
                  </button>
                </div>

                <div class="fp-chip-grid quincena">
                  <button
                    v-for="item in opcionesQuincena"
                    :key="item.value"
                    type="button"
                    class="fp-chip-card"
                    :class="{ active: isSchedulerOptionSelected(item.value) }"
                    @click="toggleSchedulerOption(item.value)"
                  >
                    {{ item.text }}
                  </button>
                </div>
              </div>

              <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'F'">
                <div class="fp-step-title">
                  <span class="fp-step-badge">2</span>
                  Selección por fechas
                </div>

                <DateRangesPicker v-model="intervaloActivo.fechasConfig" />
              </div>

              <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'X'">
                <div class="fp-step-title">
                  <span class="fp-step-badge">2</span>
                  Intervalo personalizado
                </div>

                <div class="fp-number-row">
                  <button type="button" class="fp-stepper-btn" @click="decrementarIntervaloDias">‹</button>
                  <div class="fp-stepper-value">
                    {{ reglaIntervaloDias }} {{ reglaIntervaloDias === 1 ? 'día' : 'días' }}
                  </div>
                  <button type="button" class="fp-stepper-btn" @click="incrementarIntervaloDias">›</button>
                </div>
              </div>

              <div class="fp-section" v-if="intervaloActivo">
                <div class="fp-inline-grid">
                  <div>
                    <div class="fp-step-title">
                      <span class="fp-step-badge">3</span>
                      Fecha de inicio (opcional)
                    </div>

                    <DxDateBox
                      v-model:value="reglaFechaInicioUi"
                      type="date"
                      display-format="yyyy-MM-dd"
                      picker-type="calendar"
                    />

                    <div class="fp-help mt-8">
                      Si no se especifica, la programación comenzará desde hoy.
                    </div>
                  </div>

                  <div>
                    <div class="fp-step-title">
                      <span class="fp-step-badge">4</span>
                      Hora de generación
                    </div>

                    <div class="fp-hours-wrap">
                      <button
                        v-for="hora in horasProgramacionDisponibles"
                        :key="hora"
                        type="button"
                        class="fp-hour-chip"
                        :class="{ active: horaSeleccionadaTexto === hora }"
                        @click="seleccionarHoraRapida(hora)"
                      >
                        {{ hora }}
                      </button>
                    </div>
                  </div>
                </div>
              </div>

              <div class="fp-preview-card" v-if="intervaloActivo">
                <div class="fp-preview-title">Resumen de Programación</div>
                <div class="fp-preview-text">{{ resumenProgramacionHumano }}</div>
              </div>

              <div class="fp-json-card" v-if="intervaloActivo">
                <div class="fp-json-title">Vista previa del payload</div>
                <pre>{{ JSON.stringify(payloadVistaPrevia, null, 2) }}</pre>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
.page{
  min-height:100vh;
  background:#4f4f4f;
  padding:40px;
  font-family:Inter,Arial,sans-serif
}
.modal{
  max-width:1240px;
  margin:0 auto;
  background:#2f2f2f;
  border-radius:20px;
  overflow:hidden;
  box-shadow:0 20px 50px rgba(0,0,0,.35)
}
.header{
  padding:24px 28px;
  border-bottom:1px solid rgba(255,255,255,.08)
}
.header h2{
  margin:0;
  color:#fff
}
.content{
  padding:24px
}
.grid-top{
  display:grid;
  grid-template-columns:.9fr 1.4fr;
  gap:20px
}
.card{
  padding:18px;
  border-radius:16px;
  background:rgba(255,255,255,.03);
  border:1px solid rgba(255,255,255,.08)
}
.card h3{
  margin-top:0;
  color:#fff
}
label{
  display:block;
  margin:12px 0 6px;
  color:#f3f3f3;
  font-weight:700
}
input,textarea{
  width:100%;
  box-sizing:border-box;
  padding:12px 14px;
  border-radius:12px;
  border:1px solid rgba(255,255,255,.12);
  background:rgba(255,255,255,.03);
  color:#fff
}
textarea{
  resize:vertical
}
.fp-builder-v3{
  display:flex;
  flex-direction:column;
  gap:20px
}
.fp-section{
  padding:4px 0 16px 0;
  border-bottom:1px solid rgba(120,120,140,.15)
}
.fp-label{
  font-size:14px;
  font-weight:700;
  margin-bottom:6px;
  color:#f3f3f3
}
.fp-help{
  font-size:12px;
  opacity:.82;
  line-height:1.45;
  color:#d4d7df
}
.mt-8{
  margin-top:8px
}
.fp-pills-wrap{
  display:flex;
  flex-wrap:wrap;
  gap:10px;
  margin-top:12px
}
.fp-mini-pill{
  border:1px solid rgba(255,255,255,.14);
  background:rgba(255,255,255,.03);
  color:#f3f3f3;
  border-radius:999px;
  min-height:38px;
  padding:0 14px;
  font-weight:600
}
.fp-mini-pill.active{
  border-color:#c784ff;
  background:rgba(176,80,255,.18)
}
.fp-mini-pill-editable{
  display:inline-flex;
  align-items:center;
  gap:8px
}
.fp-mini-pill-hint{
  font-size:11px;
  opacity:.7
}
.fp-pill-remove{
  width:18px;
  height:18px;
  display:inline-flex;
  align-items:center;
  justify-content:center;
  border-radius:999px;
  background:rgba(255,255,255,.12);
  font-size:13px
}
.fp-add-row{
  margin-top:12px
}
.fp-outline-btn{
  border-radius:12px;
  border:1px solid #c784ff;
  background:transparent;
  color:#d88eff;
  padding:10px 14px;
  font-weight:700
}
.fp-step-title{
  display:flex;
  align-items:center;
  gap:10px;
  margin-bottom:14px;
  font-size:15px;
  font-weight:700;
  color:#f3f3f3
}
.fp-step-badge{
  width:28px;
  height:28px;
  border-radius:999px;
  background:#8a2be2;
  color:#fff;
  display:inline-flex;
  align-items:center;
  justify-content:center;
  font-weight:700
}
.fp-frequency-grid{
  display:grid;
  grid-template-columns:repeat(2,minmax(0,1fr));
  gap:14px
}
.fp-frequency-card{
  min-height:86px;
  padding:18px;
  border-radius:16px;
  border:1px solid rgba(255,255,255,.12);
  background:rgba(255,255,255,.03);
  text-align:left;
  transition:.18s ease;
  color:#fff
}
.fp-frequency-card strong{
  display:block;
  font-size:15px;
  margin-bottom:6px;
  color:#fff
}
.fp-frequency-card span{
  font-size:13px;
  color:#d9dbe1
}
.fp-frequency-card.active{
  border:2px solid #b050ff;
  background:rgba(176,80,255,.12);
  box-shadow:0 0 0 3px rgba(176,80,255,.08)
}
.fp-chip-grid{
  display:grid;
  gap:12px
}
.fp-chip-grid.days{
  grid-template-columns:repeat(7,minmax(0,1fr))
}
.fp-chip-grid.quincena{
  grid-template-columns:repeat(2,minmax(0,1fr))
}
.fp-chip-card{
  min-height:64px;
  border-radius:14px;
  border:1px solid rgba(255,255,255,.14);
  background:rgba(255,255,255,.03);
  color:#fff;
  font-weight:600;
  display:flex;
  flex-direction:column;
  align-items:center;
  justify-content:center;
  gap:4px
}
.fp-chip-card small{
  font-size:11px;
  opacity:.75
}
.fp-chip-card.active{
  border:2px solid #b050ff;
  background:rgba(176,80,255,.12);
  color:#f3d8ff
}
.fp-quick-row{
  display:flex;
  gap:10px;
  flex-wrap:wrap;
  margin-bottom:14px
}
.fp-quick-chip{
  padding:8px 12px;
  border-radius:12px;
  border:1px solid #f0b84b;
  background:rgba(240,184,75,.08);
  color:#ffd275;
  font-weight:700
}
.fp-inline-grid{
  display:grid;
  grid-template-columns:repeat(2,minmax(0,1fr));
  gap:18px
}
.fp-number-row{
  display:flex;
  align-items:center;
  gap:10px
}
.fp-stepper-btn{
  width:38px;
  height:38px;
  border:1px solid rgba(255,255,255,.14);
  background:rgba(255,255,255,.03);
  color:#fff;
  border-radius:10px;
  font-size:22px;
  line-height:1
}
.fp-stepper-value{
  min-width:140px;
  text-align:center;
  padding:10px 14px;
  border-radius:12px;
  background:rgba(255,255,255,.03);
  border:1px solid rgba(255,255,255,.14);
  color:#fff;
  font-weight:700
}
.fp-hours-wrap{
  display:flex;
  gap:10px;
  flex-wrap:wrap
}
.fp-hour-chip{
  min-height:42px;
  padding:0 14px;
  border-radius:12px;
  border:1px solid #7bb2ff;
  background:rgba(41,121,255,.08);
  color:#b9d4ff;
  font-weight:700
}
.fp-hour-chip.active{
  background:rgba(41,121,255,.18);
  border-color:#7bb2ff;
  color:#fff
}
.fp-preview-card{
  padding:20px;
  border-radius:18px;
  background:linear-gradient(135deg,#a100ff,#7d14ff);
  color:#fff;
  box-shadow:0 12px 28px rgba(125,20,255,.18)
}
.fp-preview-title{
  font-size:16px;
  font-weight:800;
  margin-bottom:8px
}
.fp-preview-text{
  font-size:14px;
  line-height:1.6
}
.fp-json-card{
  padding:18px;
  border-radius:16px;
  background:#222;
  border:1px solid rgba(255,255,255,.08)
}
.fp-json-title{
  color:#fff;
  font-weight:800;
  margin-bottom:10px
}
.fp-json-card pre{
  margin:0;
  color:#dce7ff;
  font-size:12px;
  line-height:1.5;
  white-space:pre-wrap;
  word-break:break-word
}
@media (max-width:900px){
  .grid-top,.fp-frequency-grid,.fp-inline-grid{
    grid-template-columns:1fr
  }
  .fp-chip-grid.days{
    grid-template-columns:repeat(3,minmax(0,1fr))
  }
  .fp-chip-grid.quincena{
    grid-template-columns:1fr
  }
}
</style>