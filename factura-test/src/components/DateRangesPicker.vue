<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({
      rangos: [],
      reglas: []
    })
  }
})

const emit = defineEmits(['update:modelValue'])

const nombresMeses = [
  'Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo', 'Junio',
  'Julio', 'Agosto', 'Septiembre', 'Octubre', 'Noviembre', 'Diciembre'
]

const nombresDiasSemana = ['L', 'M', 'X', 'J', 'V', 'S', 'D']

const calendarioVisible = ref(inicioDeMes(new Date()))
const fechaAncla = ref(null)
const hoveredDateIso = ref(null)

const repetirMesesCantidad = ref(1)
const repetirModo = ref('count') // count | until | forever
const repetirHastaFecha = ref('')

const notificacion = ref('')

const internalValue = ref(clonarConfig(props.modelValue))

watch(
  () => props.modelValue,
  value => {
    internalValue.value = clonarConfig(value)
  },
  { deep: true }
)

watch(
  internalValue,
  value => {
    emit('update:modelValue', clonarConfig(value))
  },
  { deep: true }
)

const rangos = computed({
  get: () => {
    asegurarEstructura()
    return internalValue.value.rangos
  },
  set: v => {
    asegurarEstructura()
    internalValue.value.rangos = Array.isArray(v) ? v : []
  }
})

const reglas = computed({
  get: () => {
    asegurarEstructura()
    return internalValue.value.reglas
  },
  set: v => {
    asegurarEstructura()
    internalValue.value.reglas = Array.isArray(v) ? v : []
  }
})

const mesVisibleTitulo = computed(() =>
  `${nombresMeses[calendarioVisible.value.getMonth()]} ${calendarioVisible.value.getFullYear()}`
)

const diasCalendarioVisible = computed(() => {
  const base = calendarioVisible.value
  const primerDiaMes = inicioDeMes(base)
  const indiceInicio = convertirDiaSemanaLunesPrimero(primerDiaMes.getDay())
  const inicioGrid = sumarDias(primerDiaMes, -indiceInicio)

  return Array.from({ length: 42 }, (_, index) => {
    const fecha = sumarDias(inicioGrid, index)
    const iso = dateToIso(fecha)
    const esDelMesVisible = fecha.getMonth() === base.getMonth()
    const esHoy = iso === dateToIso(new Date())
    const estaSeleccionado = isDateSelected(iso)
    const esAncla = fechaAncla.value === iso
    const estaEnRangoTemporal = isInPreviewRange(iso)

    return {
      iso,
      label: fecha.getDate(),
      esDelMesVisible,
      esHoy,
      estaSeleccionado,
      esAncla,
      estaEnRangoTemporal
    }
  })
})

const resumenCompacto = computed(() => {
  const itemsRangos = rangos.value.map((rango, index) => ({
    id: `r-${rango.inicio}-${rango.fin ?? 'null'}-${index}`,
    text: getTextoRangoCompacto(rango),
    full: getTextoRango(rango),
    type: 'rango',
    index
  }))

  const itemsReglas = reglas.value.map((regla, index) => ({
    id: `g-${regla.tipo}-${index}`,
    text: getTextoReglaCompacto(regla),
    full: getTextoRegla(regla),
    type: 'regla',
    index
  }))

  return [...itemsRangos, ...itemsReglas]
})

function asegurarEstructura() {
  if (!internalValue.value || typeof internalValue.value !== 'object') {
    internalValue.value = { rangos: [], reglas: [] }
  }

  if (!Array.isArray(internalValue.value.rangos)) {
    internalValue.value.rangos = []
  }

  if (!Array.isArray(internalValue.value.reglas)) {
    internalValue.value.reglas = []
  }
}

function clonarConfig(value) {
  return {
    rangos: Array.isArray(value?.rangos)
      ? value.rangos.map(r => ({
          inicio: r.inicio ?? null,
          fin: r.fin ?? null
        }))
      : [],
    reglas: Array.isArray(value?.reglas)
      ? value.reglas.map(r => ({ ...r }))
      : []
  }
}

function limpiarSeleccionTemporalCalendario() {
  fechaAncla.value = null
  hoveredDateIso.value = null
}

function limpiarNotificacion() {
  notificacion.value = ''
}

function calendarioMesAnterior() {
  calendarioVisible.value = addMonths(calendarioVisible.value, -1)
}

function calendarioMesSiguiente() {
  calendarioVisible.value = addMonths(calendarioVisible.value, 1)
}

function irAHoyCalendario() {
  calendarioVisible.value = inicioDeMes(new Date())
}

function onDayMouseEnter(iso) {
  hoveredDateIso.value = iso
}

function onDayMouseLeave() {
  hoveredDateIso.value = null
}

function onCalendarDayClick(iso) {
  limpiarNotificacion()

  if (!fechaAncla.value) {
    fechaAncla.value = iso
    return
  }

  agregarRango(fechaAncla.value, iso)
  limpiarSeleccionTemporalCalendario()
}

function agregarDiaUnicoDesdeAncla() {
  limpiarNotificacion()
  if (!fechaAncla.value) return
  agregarRango(fechaAncla.value, fechaAncla.value)
  limpiarSeleccionTemporalCalendario()
}

function agregarDesdeFechaAncla() {
  limpiarNotificacion()
  if (!fechaAncla.value) return
  insertarRango({
    inicio: fechaAncla.value,
    fin: null
  })
  limpiarSeleccionTemporalCalendario()
}

function agregarMesVisibleCompleto() {
  limpiarNotificacion()
  const inicio = dateToIso(inicioDeMes(calendarioVisible.value))
  const fin = dateToIso(finDeMes(calendarioVisible.value))
  insertarRango({ inicio, fin })
  limpiarSeleccionTemporalCalendario()
}

function limpiarTodo() {
  rangos.value = []
  reglas.value = []
  limpiarSeleccionTemporalCalendario()
  limpiarNotificacion()
}

function eliminarRango(index) {
  rangos.value.splice(index, 1)
}

function eliminarRegla(index) {
  reglas.value.splice(index, 1)
}

function eliminarItemResumen(item) {
  if (item.type === 'rango') {
    eliminarRango(item.index)
    return
  }

  eliminarRegla(item.index)
}

function agregarRango(inicioIso, finIso) {
  let inicio = inicioIso
  let fin = finIso

  if (!inicio || !fin) return

  if (compararIso(inicio, fin) > 0) {
    const tmp = inicio
    inicio = fin
    fin = tmp
  }

  insertarRango({ inicio, fin })
}

function insertarRango(nuevo) {
  const lista = rangos.value || []

  const existe = lista.some(item =>
    item.inicio === nuevo.inicio &&
    item.fin === nuevo.fin
  )

  if (existe) return

  lista.push({
    inicio: nuevo.inicio,
    fin: nuevo.fin
  })

  lista.sort(ordenarRangos)
}

function insertarRegla(nueva) {
  const lista = reglas.value || []

  const firmaNueva = JSON.stringify(nueva)
  const existe = lista.some(item => JSON.stringify(item) === firmaNueva)

  if (existe) return

  lista.push({ ...nueva })
}

function agregarDiaDelMes(dayNumber) {
  limpiarNotificacion()

  const fechaBase = calendarioVisible.value
  const ultimo = finDeMes(fechaBase).getDate()
  const ajustado = Math.min(dayNumber, ultimo)

  if (ajustado !== dayNumber) {
    notificacion.value = `El día ${dayNumber} no existe en ${nombresMeses[fechaBase.getMonth()]} ${fechaBase.getFullYear()}. Se ajustó a ${ajustado}.`
  }

  const fecha = new Date(fechaBase.getFullYear(), fechaBase.getMonth(), ajustado)
  const iso = dateToIso(fecha)
  insertarRango({ inicio: iso, fin: iso })
}

function agregarUltimoDiaMesVisible() {
  limpiarNotificacion()
  const fecha = finDeMes(calendarioVisible.value)
  const iso = dateToIso(fecha)
  insertarRango({ inicio: iso, fin: iso })
}

function toggleWeekdayEnMesVisible(weekdayIndex) {
  limpiarNotificacion()

  const dias = obtenerDiasDelMesVisiblePorWeekday(weekdayIndex)
  if (!dias.length) return

  const todosExisten = dias.every(iso =>
    rangos.value.some(r => r.inicio === iso && r.fin === iso)
  )

  if (todosExisten) {
    dias.forEach(iso => {
      const idx = rangos.value.findIndex(r => r.inicio === iso && r.fin === iso)
      if (idx >= 0) rangos.value.splice(idx, 1)
    })
    return
  }

  dias.forEach(iso => {
    insertarRango({ inicio: iso, fin: iso })
  })
}

function isWeekdayFullySelected(weekdayIndex) {
  const dias = obtenerDiasDelMesVisiblePorWeekday(weekdayIndex)
  if (!dias.length) return false

  return dias.every(iso =>
    rangos.value.some(r => r.inicio === iso && r.fin === iso)
  )
}

function seleccionarDiasHabilesMesVisible() {
  limpiarNotificacion()

  const year = calendarioVisible.value.getFullYear()
  const month = calendarioVisible.value.getMonth()
  const ultimoDia = finDeMes(calendarioVisible.value).getDate()

  for (let day = 1; day <= ultimoDia; day++) {
    const fecha = new Date(year, month, day)
    const weekday = convertirDomingoASabado(fecha.getDay())
    if (weekday >= 0 && weekday <= 4) {
      const iso = dateToIso(fecha)
      insertarRango({ inicio: iso, fin: iso })
    }
  }
}

function seleccionarFinDeSemanaMesVisible() {
  limpiarNotificacion()

  const year = calendarioVisible.value.getFullYear()
  const month = calendarioVisible.value.getMonth()
  const ultimoDia = finDeMes(calendarioVisible.value).getDate()

  for (let day = 1; day <= ultimoDia; day++) {
    const fecha = new Date(year, month, day)
    const weekday = convertirDomingoASabado(fecha.getDay())
    if (weekday === 5 || weekday === 6) {
      const iso = dateToIso(fecha)
      insertarRango({ inicio: iso, fin: iso })
    }
  }
}

function agregarReglaMensualDia(dayNumber, label) {
  limpiarNotificacion()

  const monthIso = `${calendarioVisible.value.getFullYear()}-${String(calendarioVisible.value.getMonth() + 1).padStart(2, '0')}`

  insertarRegla({
    tipo: 'monthly_day',
    day: dayNumber,
    desdeMes: monthIso,
    hasta: null,
    label
  })
}

function agregarReglaUltimoDiaMensual() {
  limpiarNotificacion()

  const monthIso = `${calendarioVisible.value.getFullYear()}-${String(calendarioVisible.value.getMonth() + 1).padStart(2, '0')}`

  insertarRegla({
    tipo: 'monthly_last_day',
    desdeMes: monthIso,
    hasta: null,
    label: 'Último día de cada mes'
  })
}

function agregarReglaDiasHabilesMensual() {
  limpiarNotificacion()

  const monthIso = `${calendarioVisible.value.getFullYear()}-${String(calendarioVisible.value.getMonth() + 1).padStart(2, '0')}`

  insertarRegla({
    tipo: 'monthly_business_days',
    desdeMes: monthIso,
    hasta: null,
    label: 'Todos los días hábiles de cada mes'
  })
}

function repetirMesVisible() {
  limpiarNotificacion()

  const year = calendarioVisible.value.getFullYear()
  const month = calendarioVisible.value.getMonth()
  const inicioMesBase = inicioDeMes(calendarioVisible.value)
  const finIsoMesBase = dateToIso(finDeMes(calendarioVisible.value))
  const inicioIsoMesBase = dateToIso(inicioMesBase)

  const rangosDelMesBase = rangos.value.filter(r => {
    const finComparacion = r.fin == null ? r.inicio : r.fin
    return !(finComparacion < inicioIsoMesBase || r.inicio > finIsoMesBase)
  })

  if (!rangosDelMesBase.length) {
    notificacion.value = 'No hay rangos dentro del mes visible para repetir.'
    return
  }

  if (repetirModo.value === 'forever') {
    const patrones = construirPatronesDesdeRangosMes(rangosDelMesBase)
    patrones.forEach(p => insertarRegla(p))
    notificacion.value = 'Se creó una regla recurrente indefinida a partir del mes visible.'
    return
  }

  if (repetirModo.value === 'until') {
    if (!repetirHastaFecha.value) {
      notificacion.value = 'Selecciona una fecha límite.'
      return
    }

    const hasta = new Date(repetirHastaFecha.value)
    if (Number.isNaN(hasta.getTime())) {
      notificacion.value = 'La fecha límite no es válida.'
      return
    }

    const meses = calcularMesesHasta(year, month, hasta)
    if (meses < 1) {
      notificacion.value = 'La fecha límite debe estar después del mes visible.'
      return
    }

    replicarRangosMes(rangosDelMesBase, year, month, meses)
    return
  }

  const cantidad = Number(repetirMesesCantidad.value || 0)
  if (cantidad < 1) {
    notificacion.value = 'Indica cuántos meses siguientes quieres replicar.'
    return
  }

  replicarRangosMes(rangosDelMesBase, year, month, cantidad)
}

function construirPatronesDesdeRangosMes(rangosDelMesBase) {
  const salida = []
  const year = calendarioVisible.value.getFullYear()
  const month = calendarioVisible.value.getMonth()
  const monthIso = `${year}-${String(month + 1).padStart(2, '0')}`
  const inicioMesVisible = inicioDeMes(calendarioVisible.value)
  const finMesVisible = finDeMes(calendarioVisible.value)

  rangosDelMesBase.forEach(r => {
    if (!r.fin || r.inicio === r.fin) {
      const fecha = isoToDate(r.inicio)
      const day = fecha.getDate()

      salida.push({
        tipo: 'monthly_day',
        day,
        desdeMes: monthIso,
        hasta: null,
        label: `Dia ${day} de cada mes`
      })
      return
    }

    const fechaInicio = isoToDate(r.inicio)
    const fechaFin = isoToDate(r.fin)
    const inicioAjustado = fechaInicio < inicioMesVisible ? inicioMesVisible : fechaInicio
    const finAjustado = fechaFin > finMesVisible ? finMesVisible : fechaFin

    for (let day = inicioAjustado.getDate(); day <= finAjustado.getDate(); day++) {
      salida.push({
        tipo: 'monthly_day',
        day,
        desdeMes: monthIso,
        hasta: null,
        label: `Dia ${day} de cada mes`
      })
    }
  })

  return salida.filter((regla, index, reglas) =>
    reglas.findIndex(item =>
      item.tipo === regla.tipo &&
      item.day === regla.day &&
      item.desdeMes === regla.desdeMes &&
      item.hasta === regla.hasta
    ) === index
  )
}

function replicarRangosMes(rangosDelMesBase, year, month, cantidad) {
  const avisos = []

  for (let offset = 1; offset <= cantidad; offset++) {
    const destinoBase = new Date(year, month + offset, 1)
    const ultimoDiaDestino = finDeMes(destinoBase).getDate()

    rangosDelMesBase.forEach(rango => {
      const inicioDate = isoToDate(rango.inicio)
      const finDate = rango.fin ? isoToDate(rango.fin) : null

      const inicioDiaOriginal = inicioDate.getDate()
      const inicioAjustado = Math.min(inicioDiaOriginal, ultimoDiaDestino)

      const nuevoInicio = new Date(
        destinoBase.getFullYear(),
        destinoBase.getMonth(),
        inicioAjustado
      )

      let nuevoFin = null
      let huboAjuste = inicioAjustado !== inicioDiaOriginal

      if (rango.fin != null) {
        const finDiaOriginal = finDate.getDate()
        const finAjustado = Math.min(finDiaOriginal, ultimoDiaDestino)

        nuevoFin = new Date(
          destinoBase.getFullYear(),
          destinoBase.getMonth(),
          finAjustado
        )

        if (finAjustado !== finDiaOriginal) {
          huboAjuste = true
        }

        if (nuevoFin < nuevoInicio) {
          nuevoFin = new Date(nuevoInicio)
        }
      }

      insertarRango({
        inicio: dateToIso(nuevoInicio),
        fin: nuevoFin ? dateToIso(nuevoFin) : null
      })

      if (huboAjuste) {
        avisos.push(
          `${nombresMeses[destinoBase.getMonth()]} ${destinoBase.getFullYear()} fue ajustado al último día válido`
        )
      }
    })
  }

  if (avisos.length) {
    notificacion.value = [...new Set(avisos)].join(' · ')
  } else {
    notificacion.value = 'Se replicó el mes visible.'
  }
}

function obtenerDiasDelMesVisiblePorWeekday(weekdayIndex) {
  const year = calendarioVisible.value.getFullYear()
  const month = calendarioVisible.value.getMonth()
  const ultimoDia = finDeMes(calendarioVisible.value).getDate()
  const salida = []

  for (let day = 1; day <= ultimoDia; day++) {
    const fecha = new Date(year, month, day)
    const weekday = convertirDomingoASabado(fecha.getDay())
    if (weekday === weekdayIndex) {
      salida.push(dateToIso(fecha))
    }
  }

  return salida
}

function calcularMesesHasta(year, month, hastaDate) {
  const targetYear = hastaDate.getFullYear()
  const targetMonth = hastaDate.getMonth()
  return (targetYear - year) * 12 + (targetMonth - month)
}

function ordenarRangos(a, b) {
  if (a.inicio < b.inicio) return -1
  if (a.inicio > b.inicio) return 1

  const finA = a.fin == null ? '9999-12-31' : a.fin
  const finB = b.fin == null ? '9999-12-31' : b.fin

  if (finA < finB) return -1
  if (finA > finB) return 1
  return 0
}

function isDateSelected(iso) {
  return rangos.value.some(r => isoDentroDeRango(iso, r))
}

function isInPreviewRange(iso) {
  if (!fechaAncla.value || !hoveredDateIso.value) return false

  let inicio = fechaAncla.value
  let fin = hoveredDateIso.value

  if (compararIso(inicio, fin) > 0) {
    const tmp = inicio
    inicio = fin
    fin = tmp
  }

  return iso >= inicio && iso <= fin
}

function isoDentroDeRango(iso, rango) {
  if (!rango) return false

  if (rango.fin == null || rango.fin === '') {
    return iso >= rango.inicio
  }

  return iso >= rango.inicio && iso <= rango.fin
}

function getTextoRango(rango) {
  if (!rango) return ''

  if (rango.fin == null || rango.fin === '') {
    return `desde ${formatoFechaLargaIso(rango.inicio)}`
  }

  if (rango.inicio === rango.fin) {
    return formatoFechaLargaIso(rango.inicio)
  }

  return `${formatoFechaLargaIso(rango.inicio)} al ${formatoFechaLargaIso(rango.fin)}`
}

function getTextoRangoCompacto(rango) {
  if (!rango) return ''

  if (rango.fin == null || rango.fin === '') {
    return `Desde ${formatoFechaMini(rango.inicio)}`
  }

  if (rango.inicio === rango.fin) {
    return formatoFechaMini(rango.inicio)
  }

  return `${formatoFechaMini(rango.inicio)} → ${formatoFechaMini(rango.fin)}`
}

function getTextoRegla(regla) {
  if (!regla) return ''

  if (regla.tipo === 'monthly_day') {
    return `${regla.label || `Día ${regla.day} de cada mes`} · desde ${regla.desdeMes}`
  }

  if (regla.tipo === 'monthly_last_day') {
    return `${regla.label || 'Último día de cada mes'} · desde ${regla.desdeMes}`
  }

  if (regla.tipo === 'monthly_business_days') {
    return `${regla.label || 'Días hábiles del mes'} · desde ${regla.desdeMes}`
  }

  if (regla.tipo === 'monthly_range') {
    return `${regla.label || `Días ${regla.dayStart}-${regla.dayEnd} de cada mes`} · desde ${regla.desdeMes}`
  }

  return 'Regla'
}

function getTextoReglaCompacto(regla) {
  if (!regla) return ''

  if (regla.tipo === 'monthly_day') {
    return `Cada mes: día ${regla.day}`
  }

  if (regla.tipo === 'monthly_last_day') {
    return 'Cada mes: último día'
  }

  if (regla.tipo === 'monthly_business_days') {
    return 'Cada mes: hábiles'
  }

  if (regla.tipo === 'monthly_range') {
    return `Cada mes: ${regla.dayStart}-${regla.dayEnd}`
  }

  return 'Regla'
}

function formatoFechaLargaIso(iso) {
  if (!iso) return ''
  const fecha = isoToDate(iso)
  return fecha.toLocaleDateString('es-MX', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric'
  })
}

function formatoFechaMini(iso) {
  if (!iso) return ''
  const fecha = isoToDate(iso)
  return fecha.toLocaleDateString('es-MX', {
    day: '2-digit',
    month: '2-digit'
  })
}

function inicioDeMes(date) {
  return new Date(date.getFullYear(), date.getMonth(), 1)
}

function finDeMes(date) {
  return new Date(date.getFullYear(), date.getMonth() + 1, 0)
}

function addMonths(date, diff) {
  return new Date(date.getFullYear(), date.getMonth() + diff, 1)
}

function sumarDias(date, dias) {
  return new Date(date.getFullYear(), date.getMonth(), date.getDate() + dias)
}

function convertirDiaSemanaLunesPrimero(dayJsStyle) {
  return dayJsStyle === 0 ? 6 : dayJsStyle - 1
}

function convertirDomingoASabado(dayJsStyle) {
  return dayJsStyle === 0 ? 6 : dayJsStyle - 1
}

function dateToIso(date) {
  const y = date.getFullYear()
  const m = String(date.getMonth() + 1).padStart(2, '0')
  const d = String(date.getDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}

function isoToDate(iso) {
  const [y, m, d] = String(iso).split('-').map(Number)
  return new Date(y, (m || 1) - 1, d || 1)
}

function compararIso(a, b) {
  if (a < b) return -1
  if (a > b) return 1
  return 0
}
</script>

<template>
  <div class="drp">
    <div class="drp-head">
      <div class="drp-title-wrap">
        <div class="drp-title">Selección por fechas</div>
        <div class="drp-help">
          Primer click = inicio. Segundo click = fin. Si repites el mismo día, se guarda como rango de un solo día.
        </div>
      </div>

      <div class="drp-nav">
        <button type="button" class="drp-nav-btn" @click="calendarioMesAnterior">‹</button>
        <div class="drp-nav-label">{{ mesVisibleTitulo }}</div>
        <button type="button" class="drp-nav-btn" @click="calendarioMesSiguiente">›</button>
      </div>
    </div>

    <div class="drp-layout">
      <div class="drp-main">
        <div class="drp-actions">
          <button type="button" class="drp-chip" @click="irAHoyCalendario">Hoy</button>
          <button type="button" class="drp-chip" @click="agregarMesVisibleCompleto">Mes visible</button>
          <button type="button" class="drp-chip" :disabled="!fechaAncla" @click="agregarDiaUnicoDesdeAncla">
            Solo día
          </button>
          <button type="button" class="drp-chip" :disabled="!fechaAncla" @click="agregarDesdeFechaAncla">
            Desde fecha
          </button>
          <button type="button" class="drp-chip danger" @click="limpiarTodo">
            Limpiar
          </button>
        </div>

        <div class="drp-actions">
          <button type="button" class="drp-chip" @click="agregarDiaDelMes(1)">Día 1</button>
          <button type="button" class="drp-chip" @click="agregarDiaDelMes(15)">Día 15</button>
          <button type="button" class="drp-chip" @click="agregarUltimoDiaMesVisible">Último día</button>
          <button type="button" class="drp-chip primary" @click="agregarReglaMensualDia(1, 'Día 1 de cada mes')">
            Día 1 cada mes
          </button>
          <button type="button" class="drp-chip primary" @click="agregarReglaMensualDia(15, 'Día 15 de cada mes')">
            Día 15 cada mes
          </button>
          <button type="button" class="drp-chip primary" @click="agregarReglaUltimoDiaMensual">
            Último día cada mes
          </button>
          <button type="button" class="drp-chip primary" @click="agregarReglaDiasHabilesMensual">
            Hábiles cada mes
          </button>
        </div>

        <div class="drp-repeat-bar">
          <div class="drp-repeat-title">Repetir mes visible</div>

          <div class="drp-repeat-mode-row">
            <label class="drp-radio">
              <input v-model="repetirModo" type="radio" value="count" />
              <span>N meses</span>
            </label>

            <label class="drp-radio">
              <input v-model="repetirModo" type="radio" value="until" />
              <span>Hasta fecha</span>
            </label>

            <label class="drp-radio">
              <input v-model="repetirModo" type="radio" value="forever" />
              <span>Indefinido</span>
            </label>
          </div>

          <div class="drp-repeat-controls">
            <template v-if="repetirModo === 'count'">
              <input
                v-model.number="repetirMesesCantidad"
                type="number"
                min="1"
                max="60"
                class="drp-repeat-input"
              />
              <span class="drp-repeat-text">mes(es) siguientes</span>
            </template>

            <template v-if="repetirModo === 'until'">
              <input
                v-model="repetirHastaFecha"
                type="date"
                class="drp-repeat-input date"
              />
            </template>

            <template v-if="repetirModo === 'forever'">
              <span class="drp-repeat-text">Se guarda como regla recurrente mensual</span>
            </template>

            <button type="button" class="drp-chip primary" @click="repetirMesVisible">
              Repetir
            </button>
          </div>
        </div>

        <div class="drp-calendar-card">
          <div class="drp-calendar-top-actions">
            <button type="button" class="drp-mini-chip" @click="seleccionarDiasHabilesMesVisible">
              Hábiles
            </button>

            <button type="button" class="drp-mini-chip" @click="seleccionarFinDeSemanaMesVisible">
              Fin de semana
            </button>
          </div>

          <div class="drp-weekdays drp-weekdays-interactive">
            <button
              v-for="(dia, index) in nombresDiasSemana"
              :key="dia"
              type="button"
              class="drp-weekday drp-weekday-btn"
              :class="{ active: isWeekdayFullySelected(index) }"
              :title="`Seleccionar todos los ${dia} del mes visible`"
              @click="toggleWeekdayEnMesVisible(index)"
            >
              {{ dia }}
            </button>
          </div>

          <div class="drp-grid">
            <button
              v-for="day in diasCalendarioVisible"
              :key="day.iso"
              type="button"
              class="drp-day"
              :class="{
                muted: !day.esDelMesVisible,
                today: day.esHoy,
                active: day.estaSeleccionado,
                anchor: day.esAncla,
                preview: day.estaEnRangoTemporal
              }"
              @mouseenter="onDayMouseEnter(day.iso)"
              @mouseleave="onDayMouseLeave"
              @click="onCalendarDayClick(day.iso)"
            >
              {{ day.label }}
            </button>
          </div>
        </div>

        <div v-if="fechaAncla" class="drp-note">
          Inicio seleccionado: <strong>{{ formatoFechaLargaIso(fechaAncla) }}</strong>
        </div>

        <div v-if="notificacion" class="drp-note info">
          {{ notificacion }}
        </div>
      </div>

      <aside class="drp-side">
        <div class="drp-side-head">
          <strong>Selecciones</strong>
          <span>{{ resumenCompacto.length }}</span>
        </div>

        <div v-if="resumenCompacto.length" class="drp-side-list">
          <div
            v-for="item in resumenCompacto"
            :key="item.id"
            class="drp-side-item"
            :title="item.full"
          >
            <div class="drp-side-text">{{ item.text }}</div>
            <button type="button" class="drp-side-remove" @click="eliminarItemResumen(item)">
              ×
            </button>
          </div>
        </div>

        <div v-else class="drp-empty">
          Sin rangos ni reglas
        </div>
      </aside>
    </div>
  </div>
</template>

<style scoped>
.drp{
  display:flex;
  flex-direction:column;
  gap:12px;
  color:#fff
}
.drp-head{
  display:flex;
  justify-content:space-between;
  align-items:flex-start;
  gap:12px;
  flex-wrap:wrap
}
.drp-title{
  font-size:14px;
  font-weight:800
}
.drp-help{
  margin-top:4px;
  font-size:11px;
  color:#d4d7df;
  line-height:1.4;
  max-width:620px
}
.drp-nav{
  display:flex;
  align-items:center;
  gap:8px
}
.drp-nav-btn{
  width:32px;
  height:32px;
  border-radius:10px;
  border:1px solid rgba(255,255,255,.14);
  background:rgba(255,255,255,.03);
  color:#fff;
  font-size:20px;
  line-height:1
}
.drp-nav-label{
  min-width:170px;
  text-align:center;
  padding:8px 12px;
  border-radius:10px;
  border:1px solid rgba(255,255,255,.12);
  background:rgba(255,255,255,.03);
  font-size:13px;
  font-weight:700
}
.drp-layout{
  display:grid;
  grid-template-columns:minmax(0,1fr) 250px;
  gap:14px
}
.drp-main{
  min-width:0
}
.drp-actions{
  display:flex;
  flex-wrap:wrap;
  gap:8px;
  margin-bottom:10px
}
.drp-chip{
  padding:7px 10px;
  border-radius:10px;
  border:1px solid #f0b84b;
  background:rgba(240,184,75,.08);
  color:#ffd275;
  font-size:12px;
  font-weight:700
}
.drp-chip.primary{
  border-color:#7bb2ff;
  background:rgba(41,121,255,.1);
  color:#d5e7ff
}
.drp-chip.danger{
  border-color:#ff8c8c;
  color:#ffb5b5;
  background:rgba(255,90,90,.08)
}
.drp-chip:disabled{
  opacity:.45;
  cursor:not-allowed
}
.drp-repeat-bar{
  display:flex;
  flex-direction:column;
  gap:10px;
  padding:10px 12px;
  margin-bottom:10px;
  border-radius:12px;
  background:rgba(255,255,255,.03);
  border:1px solid rgba(255,255,255,.08)
}
.drp-repeat-title{
  font-size:12px;
  font-weight:700;
  color:#f3f3f3
}
.drp-repeat-mode-row{
  display:flex;
  gap:14px;
  flex-wrap:wrap
}
.drp-radio{
  display:flex;
  align-items:center;
  gap:6px;
  font-size:12px;
  color:#d4d7df
}
.drp-repeat-controls{
  display:flex;
  align-items:center;
  gap:8px;
  flex-wrap:wrap
}
.drp-repeat-input{
  width:86px;
  padding:8px 10px;
  border-radius:10px;
  border:1px solid rgba(255,255,255,.12);
  background:rgba(255,255,255,.03);
  color:#fff
}
.drp-repeat-input.date{
  width:150px
}
.drp-repeat-text{
  font-size:12px;
  color:#d4d7df
}
.drp-calendar-card{
  padding:12px;
  border-radius:14px;
  background:rgba(255,255,255,.03);
  border:1px solid rgba(255,255,255,.12)
}
.drp-calendar-top-actions{
  display:flex;
  justify-content:flex-end;
  gap:8px;
  flex-wrap:wrap;
  margin-bottom:8px
}
.drp-mini-chip{
  padding:6px 10px;
  border-radius:9px;
  border:1px solid #f0b84b;
  background:rgba(240,184,75,.08);
  color:#ffd275;
  font-size:11px;
  font-weight:700
}
.drp-weekdays{
  display:grid;
  grid-template-columns:repeat(7,minmax(0,1fr));
  gap:6px;
  margin-bottom:6px
}
.drp-weekdays-interactive{
  margin-bottom:6px
}
.drp-weekday{
  text-align:center;
  color:#d4d7df;
  font-size:11px;
  font-weight:700;
  padding:4px 0
}
.drp-weekday-btn{
  border:none;
  border-radius:10px;
  background:rgba(176,80,255,.08);
  border:1px solid rgba(176,80,255,.35);
  color:#f3d8ff;
  cursor:pointer;
  min-height:30px;
  transition:.15s ease
}
.drp-weekday-btn:hover{
  background:rgba(176,80,255,.16);
  border-color:#b050ff
}
.drp-weekday-btn.active{
  background:rgba(176,80,255,.22);
  border-color:#b050ff;
  color:#fff
}
.drp-grid{
  display:grid;
  grid-template-columns:repeat(7,minmax(0,1fr));
  gap:6px
}
.drp-day{
  min-height:42px;
  border-radius:12px;
  border:1px solid rgba(255,255,255,.12);
  background:rgba(255,255,255,.03);
  color:#fff;
  font-size:12px;
  font-weight:700
}
.drp-day.muted{
  opacity:.35
}
.drp-day.today{
  box-shadow:inset 0 0 0 1px #7bb2ff
}
.drp-day.preview{
  background:rgba(240,184,75,.1);
  border-color:#f0b84b
}
.drp-day.active{
  background:rgba(176,80,255,.18);
  border:2px solid #b050ff;
  color:#f3d8ff
}
.drp-day.anchor{
  box-shadow:0 0 0 2px rgba(255,255,255,.12) inset, 0 0 0 2px rgba(176,80,255,.18)
}
.drp-note{
  margin-top:10px;
  padding:10px 12px;
  border-radius:10px;
  background:rgba(123,178,255,.08);
  border:1px solid rgba(123,178,255,.25);
  color:#d6e7ff;
  font-size:12px
}
.drp-note.info{
  background:rgba(240,184,75,.08);
  border-color:rgba(240,184,75,.28);
  color:#ffe0a3
}
.drp-side{
  padding:10px;
  border-radius:14px;
  background:rgba(255,255,255,.03);
  border:1px solid rgba(255,255,255,.12);
  max-height:520px;
  overflow:auto
}
.drp-side-head{
  display:flex;
  justify-content:space-between;
  align-items:center;
  margin-bottom:10px;
  font-size:12px;
  color:#f3f3f3
}
.drp-side-head span{
  color:#d4d7df
}
.drp-side-list{
  display:flex;
  flex-direction:column;
  gap:8px
}
.drp-side-item{
  display:flex;
  align-items:center;
  justify-content:space-between;
  gap:8px;
  padding:8px 10px;
  border-radius:10px;
  background:rgba(255,255,255,.03);
  border:1px solid rgba(255,255,255,.08)
}
.drp-side-text{
  font-size:12px;
  font-weight:700;
  color:#fff;
  min-width:0;
  overflow:hidden;
  text-overflow:ellipsis;
  white-space:nowrap
}
.drp-side-remove{
  width:24px;
  height:24px;
  border-radius:8px;
  border:1px solid rgba(255,120,120,.35);
  background:rgba(255,90,90,.08);
  color:#ffb5b5;
  font-weight:800;
  flex:0 0 auto
}
.drp-empty{
  padding:12px;
  border-radius:10px;
  border:1px dashed rgba(255,255,255,.12);
  color:#d4d7df;
  font-size:12px
}
@media (max-width: 980px){
  .drp-layout{
    grid-template-columns:1fr
  }
  .drp-side{
    max-height:none
  }
}
</style>

