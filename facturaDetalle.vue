<template>
    <div class="row">
        <div class="col-md-12 col-sm-12">
            <dx-data-grid
                id="grid_facturacion"
                :data-source="dataSource"
                :remote-operations="true"
                :allow-column-reordering="true"
                :row-alternation-enabled="true"
                :show-borders="true"
                :selection="{ mode: 'single' }"
                :hover-state-enabled="true"
                :column-auto-width="true"
                :height="pantallaHeight"
                :filter-value="customFilter"
                @selectionChanged="selectedChanged"
                @rowClick="onRowClick"
                @rowDblClick="onRowDblClick"
                @rowExpanding="onRowExpanding"
                @initialized="saveComponent"
                @rowInserting="grabar"
                @rowUpdating="grabar"
                @editingStart="editStart"
                @toolbar-preparing="toolBar($event)"
                @editorPreparing="editorPreparingFacturas"
                ref="gridFacturacion"
            >
                <DxEditing mode="popup">
                    <DxPopup
                        :show-title="true"
                        :width="'1180px'"
                        :height="'auto'"
                        :title="'Configuración de Factura Programada'"
                    >
                        <DxPosition my="center" at="center" of="window"/>
                        <DxToolbarItem
                            widget="dxButton"
                            :options="{ text:'Guardar', icon:'fa fa-save', type:'success', onClick: facturacionGrabar }"
                            location="after"
                            toolbar="bottom"
                        />
                        <DxToolbarItem
                            widget="dxButton"
                            :options="{ text:'Cancelar', icon:'fa fa-times', type:'danger', onClick: facturacionCerrar }"
                            location="after"
                            toolbar="bottom"
                        />
                    </DxPopup>

                    <DxForm
                        :on-content-ready="validateForm"
                        label-location="top"
                        :col-count="2"
                        css-class="factura-programada-form"
                    >
                        <dx-item item-type="group" caption="Plantilla base" :col-span="1" css-class="fp-card">
                            <dx-item
                                data-field="i01220idFacturaBase"
                                editor-type="dxNumberBox"
                                :editor-options="{
                                    min: 1,
                                    showSpinButtons: true,
                                    placeholder: 'ID de factura existente'
                                }"
                                :label="{ text: 'Factura base' }"
                                :help-text="'La programación solo guardará el id de la factura existente que servirá como plantilla.'"
                                :validation-rules="[{ type: 'required', message: 'Factura base es requerida' }]"
                            />

                            <dx-item
                                data-field="s01226referencia"
                                :label="{ text: 'Referencia base' }"
                                :validation-rules="[{ type: 'required', message: 'Referencia base es requerida' }]"
                            />

                            <dx-item
                                data-field="s01226comentarios"
                                editor-type="dxTextArea"
                                :editor-options="{
                                    autoResizeEnabled: true,
                                    minHeight: 90
                                }"
                                :label="{ text: 'Comentarios' }"
                            />
                        </dx-item>

                        <dx-item item-type="group" caption="Programación de la Factura" :col-span="1" css-class="programacion-builder fp-card">
                            <dx-item template="programacionVisual" :col-span="2" :label="{ visible: false }" />
                        </dx-item>
                    </DxForm>
                </DxEditing>

                <DxExport :enabled="true" :allow-export-selected-data="true" file-name="Facturación Programada" />
                <dx-filter-row :visible="true"/>
                <DxColumnChooser :enabled="true" :mode="'select'" />

                <dx-column data-field="i01226idFacturaProgramada" data-type="number" caption="ID" alignment="right" :allowHeaderFiltering="false" :visible="false" />
                <dx-column data-field="i01220idFacturaBase" data-type="number" caption="Factura base" alignment="right" />
                <dx-column data-field="s01226referencia" data-type="string" caption="Referencia base" alignment="left" />
                <dx-column
                    data-field="s01226intervaloGeneracion"
                    data-type="string"
                    caption="Intervalo"
                    alignment="left"
                    :lookup="{
                        dataSource: [
                            { value: 'S', text: 'Semana' },
                            { value: 'Q', text: 'Quincena' },
                            { value: 'M', text: 'Meses y días' },
                            { value: 'X', text: 'Cada X días' }
                        ],
                        displayExpr: 'text',
                        valueExpr: 'value'
                    }"
                />
                <dx-column data-field="s01226instruccionGeneracion" data-type="string" caption="Instrucción" alignment="left" :calculateDisplayValue="calculateInstrucciones" />
                <dx-column data-field="i01226horaGeneracion" data-type="string" caption="Hora programada" alignment="right" :calculateDisplayValue="calculateHoras" />
                <dx-column data-field="s01226comentarios" data-type="string" :allowFiltering="true" caption="Comentarios" alignment="left" :calculateDisplayValue="calculateComentariosVisibles" />
                <dx-column
                    data-field="i01226activo"
                    data-type="string"
                    :allowFiltering="true"
                    caption="Estatus"
                    alignment="center"
                    cell-template="cellActivo"
                    :lookup="{
                        dataSource: [
                            { text: 'Activo', value: 1 },
                            { text: 'Inactivo', value: 0 }
                        ],
                        displayExpr: 'text',
                        valueExpr: 'value'
                    }"
                />

                <dx-selection :select-all-mode="'page'" :show-check-boxes-mode="'onClick'" mode="multiple" />
                <dx-search-panel :visible="true" :highlight-case-sensitive="true" :placeholder="'Buscar'" />
                <dx-group-panel :visible="false"/>
                <dx-grouping :auto-expand-all="false"/>
                <dx-pager :info-text="'Pagina {0} de {1} ({2} Registros)'" :allowed-page-sizes="pageSizes" :show-page-size-selector="true" :show-navigaton-buttons="true" :show-info="true" />
                <dx-paging :page-size="100"/>

                <template #programacionVisual>
                    <div class="fp-builder-v3">
                        <div class="fp-section">
                            <div class="fp-label">Intervalos guardados del cliente</div>
                            <div class="fp-help">Selecciona una pill para cargar sus campos en esta configuración.</div>

                            <div class="fp-pills-wrap">
                                <button
                                    v-for="item in clienteProgramaciones"
                                    :key="item.id"
                                    type="button"
                                    class="fp-mini-pill"
                                    :class="{ active: String(datosgen.configuracionClienteActiva) === String(item.id) }"
                                    @click="changeConfiguracionClientePillManual(item.id)"
                                >
                                    {{ item.text }}
                                </button>
                            </div>

                            <div class="fp-add-row">
                                <button type="button" class="fp-outline-btn" @click="agregarIntervaloFactura">
                                    + Agregar intervalo
                                </button>
                            </div>
                        </div>

                        <div class="fp-section" v-if="intervalosFactura.length">
                            <div class="fp-label">Intervalos de esta programación</div>
                            <div class="fp-help">Selecciona una pill para editar sus campos.</div>

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
                                    <span v-if="intervalosFactura.length > 1" class="fp-pill-remove" @click.stop="eliminarIntervaloFactura(item.id)">×</span>
                                </button>
                            </div>
                        </div>

                        <div class="fp-section" v-if="intervaloActivo">
                            <div class="fp-step-title">
                                <span class="fp-step-badge">1</span>
                                Selecciona la frecuencia
                            </div>

                            <div class="fp-frequency-grid">
                                <button type="button" class="fp-frequency-card" :class="{ active: tipoPeriodicidad === 'S' }" @click="setFrecuenciaVisual('S')">
                                    <strong>Semanal</strong>
                                    <span>Días de la semana</span>
                                </button>
                                <button type="button" class="fp-frequency-card" :class="{ active: tipoPeriodicidad === 'Q' }" @click="setFrecuenciaVisual('Q')">
                                    <strong>Quincenal</strong>
                                    <span>Día 1, 15 o ambos</span>
                                </button>
                                <button type="button" class="fp-frequency-card" :class="{ active: tipoPeriodicidad === 'M' }" @click="setFrecuenciaVisual('M')">
                                    <strong>Meses y días</strong>
                                    <span>Selecciona meses y días válidos</span>
                                </button>
                                <button type="button" class="fp-frequency-card" :class="{ active: tipoPeriodicidad === 'X' }" @click="setFrecuenciaVisual('X')">
                                    <strong>Personalizado</strong>
                                    <span>Cada X días</span>
                                </button>
                            </div>
                        </div>

                        <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'S'">
                            <div class="fp-step-title"><span class="fp-step-badge">2</span>Selecciona los días de la semana</div>
                            <div class="fp-chip-grid days">
                                <button
                                    v-for="dia in opcionesDiasSemana"
                                    :key="dia.value"
                                    type="button"
                                    class="fp-chip-card"
                                    :class="{ active: isSchedulerOptionSelected(dia.value) }"
                                    @click="toggleSchedulerOption(dia.value); guardarEstadoVisibleEnIntervalo();"
                                >
                                    {{ dia.short }}
                                    <small>{{ dia.text }}</small>
                                </button>
                            </div>
                        </div>

                        <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'Q'">
                            <div class="fp-step-title"><span class="fp-step-badge">2</span>Selecciona los cortes</div>
                            <div class="fp-quick-row">
                                <button type="button" class="fp-quick-chip" @click="seleccionarQuincenaRapida(['1'])">Primer día</button>
                                <button type="button" class="fp-quick-chip" @click="seleccionarQuincenaRapida(['15'])">Día 15</button>
                                <button type="button" class="fp-quick-chip" @click="seleccionarQuincenaRapida(['1','15'])">1 y 15</button>
                            </div>
                            <div class="fp-chip-grid quincena">
                                <button
                                    v-for="item in opcionesQuincena"
                                    :key="item.value"
                                    type="button"
                                    class="fp-chip-card"
                                    :class="{ active: isSchedulerOptionSelected(item.value) }"
                                    @click="toggleSchedulerOption(item.value); guardarEstadoVisibleEnIntervalo();"
                                >
                                    {{ item.text }}
                                </button>
                            </div>
                        </div>

                        <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'M'">
                            <div class="fp-step-title"><span class="fp-step-badge">2</span>Selecciona los meses y sus días</div>

                            <div class="fp-quick-row">
                                <button type="button" class="fp-quick-chip" @click="seleccionarTodosMeses">Todos los meses</button>
                                <button type="button" class="fp-quick-chip" @click="limpiarMeses">Limpiar meses</button>
                                <button type="button" class="fp-quick-chip" @click="aplicarDiaATodosLosMeses(1)">Día 1 en todos</button>
                                <button type="button" class="fp-quick-chip" @click="aplicarDiaATodosLosMeses(15)">Día 15 en todos</button>
                                <button type="button" class="fp-quick-chip" @click="aplicarUltimoDiaATodosLosMeses">Último día de cada mes</button>
                            </div>

                            <div class="fp-months-grid">
                                <button
                                    v-for="mes in mesesCatalogo"
                                    :key="mes.value"
                                    type="button"
                                    class="fp-month-card"
                                    :class="{ active: isMesSeleccionado(mes.value) }"
                                    @click="toggleMes(mes.value)"
                                >
                                    <strong>{{ mes.short }}</strong>
                                    <span>{{ mes.text }}</span>
                                </button>
                            </div>

                            <div class="fp-month-navigator" v-if="mesesSeleccionadosActivos.length">
                                <div class="fp-stepper-wrap">
                                    <button type="button" class="fp-stepper-btn" @click="mesActivoAnterior">‹</button>
                                    <div class="fp-stepper-value month-viewer">
                                        {{ mesActivoUi.short }} - {{ mesActivoUi.text }}
                                    </div>
                                    <button type="button" class="fp-stepper-btn" @click="mesActivoSiguiente">›</button>
                                </div>
                            </div>

                            <div v-if="mesesSeleccionadosActivos.length" class="fp-month-calendar-card">
                                <div class="fp-calendar-header">
                                    <div>
                                        <strong>{{ mesActivoUi.text }}</strong>
                                        <span>Días disponibles para este mes</span>
                                    </div>
                                    <div class="fp-quick-row compact">
                                        <button type="button" class="fp-quick-chip" @click="seleccionarDiaMesActivo(1)">Día 1</button>
                                        <button type="button" class="fp-quick-chip" @click="seleccionarDiaMesActivo(15)">Día 15</button>
                                        <button type="button" class="fp-quick-chip" @click="seleccionarUltimoDiaMesActivo">Último día</button>
                                        <button type="button" class="fp-quick-chip" @click="limpiarDiasMesActivo">Limpiar</button>
                                    </div>
                                </div>

                                <div class="fp-chip-grid month-days-calendar">
                                    <button
                                        v-for="dia in diasDelMesActivo"
                                        :key="`${mesActivoUi.value}-${dia}`"
                                        type="button"
                                        class="fp-day-card"
                                        :class="{ active: isDiaSeleccionadoEnMesActivo(dia) }"
                                        @click="toggleDiaMesActivo(dia)"
                                    >
                                        {{ dia }}
                                    </button>
                                </div>
                            </div>

                            <div v-else class="fp-empty-note">
                                Selecciona uno o varios meses para habilitar su calendario.
                            </div>
                        </div>

                        <div class="fp-section" v-if="intervaloActivo && tipoPeriodicidad === 'X'">
                            <div class="fp-step-title"><span class="fp-step-badge">2</span>Intervalo personalizado</div>
                            <div class="fp-inline-grid one-col">
                                <div>
                                    <div class="fp-label">Repetir cada</div>
                                    <div class="fp-number-row">
                                        <button type="button" class="fp-stepper-btn" @click="decrementarIntervaloDias">‹</button>
                                        <div class="fp-stepper-value">{{ reglaIntervaloDias }} {{ reglaIntervaloDias === 1 ? 'día' : 'días' }}</div>
                                        <button type="button" class="fp-stepper-btn" @click="incrementarIntervaloDias">›</button>
                                    </div>
                                </div>
                            </div>
                        </div>

                        <div class="fp-section" v-if="intervaloActivo">
                            <div class="fp-inline-grid">
                                <div>
                                    <div class="fp-step-title"><span class="fp-step-badge">3</span>Fecha de inicio (opcional)</div>
                                    <DxDateBox
                                        :value="reglaFechaInicioUi"
                                        type="date"
                                        display-format="yyyy-MM-dd"
                                        pickerType="calendar"
                                        @value-changed="updateReglaFechaInicio"
                                    />
                                    <div class="fp-help mt-8">Si no se especifica, la programación comenzará desde hoy.</div>
                                </div>

                                <div>
                                    <div class="fp-step-title"><span class="fp-step-badge">4</span>Hora de generación</div>
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
                    </div>
                </template>

                <template #btnFacturasProgramadas="{}">
                    <BotonFlotante
                        :title="formatMessage('Click to see more options')"
                        icon="fa fa-plus fa-sm"
                        tipo="grid"
                        posicion="left"
                        :expandido="true"
                        :opciones="[
                            { index: 0, title: formatMessage('New') + ' Programación', icon: 'fa fa-plus fa-sm floatingAction float_nuevo', accion: openNuevo, permiso: permisos.create },
                            { index: 1, title: formatMessage('Edit') + ' Programación', icon: 'fa fa-pencil-alt fa-sm floatingAction float_editar', accion: openEditar, permiso: permisos.update },
                            { index: 2, title: formatMessage('Delete') + formatMessage('Programación'), icon: 'fa fa-ban fa-sm floatingAction float_borrar', accion: borrarProgramacion, permiso: permisos.delete },
                            { index: 4, title: formatMessage('Activar') + formatMessage('Programación'), icon: 'fa fa-check fa-sm floatingAction float_nuevo', accion: activarProgramacion, permiso: permisos.update },
                            { index: 3, title: formatMessage('Actualizar'), icon: 'fa fa-sync-alt fa-sm floatingAction float_reload', accion: ()=>{ gridFacturacion.refresh() }, permiso: true }
                        ]"
                    />
                </template>

                <template #cellActivo="{data}">
                    <div>
                        <div v-if="data.value && data.value == 1" style="width:10px;height:10px;background-color:#38c172;border-radius:50%;display:inline-block;"></div>
                        <div v-else-if="data.value == 0" style="width:10px;height:10px;background-color:#e3342f;border-radius:50%;display:inline-block;"></div>
                    </div>
                </template>
            </dx-data-grid>
        </div>

        <input type="hidden" value="e" class="ide_bd">
    </div>
</template>

<script>
import coremixin from '../../mixins/coremixin.vue';
import esMessages from 'devextreme/localization/messages/es.json';
import { locale, loadMessages, formatMessage } from 'devextreme/localization';
import { DxButton } from 'devextreme-vue';
import {
    DxDataGrid,
    DxColumn,
    DxGrouping,
    DxGroupPanel,
    DxColumnChooser,
    DxPager,
    DxPaging,
    DxSearchPanel,
    DxFilterRow,
    DxHeaderFilter,
    DxSelection,
    DxEditing,
    DxPopup,
    DxLookup,
    DxPosition,
    DxForm,
    DxExport,
    DxRequiredRule
} from 'devextreme-vue/data-grid';
import { DxItem } from 'devextreme-vue/form';
import { DxToolbarItem } from 'devextreme-vue/popup';
import DxDateBox from 'devextreme-vue/date-box';
import BotonFlotante from '../../widgets/BotonFlotante';

export default {
    props: ['permisos'],
    mixins: [coremixin],
    components: {
        DxDataGrid,
        DxColumn,
        DxColumnChooser,
        DxGrouping,
        DxGroupPanel,
        DxPager,
        DxPaging,
        DxSelection,
        DxSearchPanel,
        DxFilterRow,
        DxHeaderFilter,
        DxButton,
        DxEditing,
        DxPopup,
        DxLookup,
        DxPosition,
        DxForm,
        DxItem,
        DxExport,
        DxRequiredRule,
        DxToolbarItem,
        BotonFlotante,
        DxDateBox
    },
    data(){
        return {
            pantallaHeight: 500,
            gridFacturacion: null,
            formEdit: null,
            selectedRowIndex: -1,
            pageSizes: [100,150,200],
            customFilter: [],
            isNuevo: false,
            datosgen: {},
            clienteProgramaciones: [],
            clienteProgramacionesRaw: [],
            intervalosFactura: [],
            intervaloActivoId: null,
            tipoPeriodicidad: 'S',
            reglaIntervaloDias: 1,
            reglaCadaPeriodos: 1,
            reglaFechaInicioUi: null,
            horasProgramacionDisponibles: ['08:00','09:00','10:00','12:00','15:00','18:00'],
            mesActivoCalendario: 1,
            dataSource: this.gridSource({
                key: 'i01226idFacturaProgramada',
                ruta: this.get_root_path() + '/facturacionProgramada',
                allowNoSKip: true
            }),
            opcionesDiasSemana: [
                { text: 'Lunes', short: 'L', value: '0' },
                { text: 'Martes', short: 'M', value: '1' },
                { text: 'Miércoles', short: 'X', value: '2' },
                { text: 'Jueves', short: 'J', value: '3' },
                { text: 'Viernes', short: 'V', value: '4' },
                { text: 'Sábado', short: 'S', value: '5' },
                { text: 'Domingo', short: 'D', value: '6' }
            ],
            opcionesQuincena: [
                { text: 'Día 1 del mes', value: '1' },
                { text: 'Día 15 del mes', value: '15' }
            ],
            mesesCatalogo: [
                { value: 1, short: 'Ene', text: 'Enero' },
                { value: 2, short: 'Feb', text: 'Febrero' },
                { value: 3, short: 'Mar', text: 'Marzo' },
                { value: 4, short: 'Abr', text: 'Abril' },
                { value: 5, short: 'May', text: 'Mayo' },
                { value: 6, short: 'Jun', text: 'Junio' },
                { value: 7, short: 'Jul', text: 'Julio' },
                { value: 8, short: 'Ago', text: 'Agosto' },
                { value: 9, short: 'Sep', text: 'Septiembre' },
                { value: 10, short: 'Oct', text: 'Octubre' },
                { value: 11, short: 'Nov', text: 'Noviembre' },
                { value: 12, short: 'Dic', text: 'Diciembre' }
            ]
        };
    },
    computed: {
        intervaloActivo(){
            return this.intervalosFactura.find(item => String(item.id) === String(this.intervaloActivoId)) || null;
        },
        pillsIntervalosFactura(){
            return this.intervalosFactura.map((item, index) => ({
                id: item.id,
                text: item.nombre || `Inter ${index + 1}`,
                hint: this.getTextoPillIntervalo(item)
            }));
        },
        horaSeleccionadaTexto(){
            return this.formatoHoraMilitar(this.datosgen.horaProgramadaUi);
        },
        mesesSeleccionadosActivos(){
            if(!this.intervaloActivo || this.intervaloActivo.tipoPeriodicidad !== 'M') return [];
            return Object.keys(this.intervaloActivo.mesesConfig || {}).map(v => Number(v)).sort((a,b) => a-b);
        },
        mesActivoUi(){
            return this.mesesCatalogo.find(m => Number(m.value) === Number(this.mesActivoCalendario)) || this.mesesCatalogo[0];
        },
        diasDelMesActivo(){
            return this.getDiasDisponiblesPorMes(this.mesActivoCalendario);
        },
        resumenProgramacionHumano(){
            let item = this.intervaloActivo;
            if(!item) return 'Selecciona o crea un intervalo.';

            let inicio = item.reglaFechaInicioUi
                ? ` Inicio: ${this.formatoFechaCorta(item.reglaFechaInicioUi)}.`
                : ' Inicio: inmediato (desde hoy).';

            let hora = item.horaProgramadaUi
                ? ` Hora: ${this.formatoHoraCorta(item.horaProgramadaUi)}.`
                : ' Hora: sin definir.';

            if(item.tipoPeriodicidad === 'S'){
                let dias = this.getTextoInstruccion('S', item.s01226instruccionGeneracion) || 'sin días seleccionados';
                return `Se generará semanalmente los ${dias}.${inicio}${hora}`;
            }

            if(item.tipoPeriodicidad === 'Q'){
                let dias = this.getTextoInstruccion('Q', item.s01226instruccionGeneracion) || 'sin cortes seleccionados';
                return `Se generará quincenalmente en ${dias}.${inicio}${hora}`;
            }

            if(item.tipoPeriodicidad === 'M'){
                let descripcion = this.getResumenMesesYDias(item);
                return `Se generará por meses y días: ${descripcion}.${inicio}${hora}`;
            }

            return `Se generará cada ${item.reglaIntervaloDias || 1} día(s).${inicio}${hora}`;
        }
    },
    created(){
        this.pantallaHeight = $('#pantalla').height() - 30;
        this.inicializarIntervalosFactura();
    },
    mounted(){
        this.handleResize([{ element: this.$refs.gridFacturacion.$el, minusHeight: 50 }]);
    },
    methods: {
        formatMessage,
        saveComponent(e){
            if(e.element.id === 'grid_facturacion'){
                this.gridFacturacion = e.component;
            }
        },
        validateForm(e){
            this.formEdit = e.component;
            e.component.validate();
        },
        toolBar(e){
            e.toolbarOptions.items.unshift({ location: 'after', template: 'btnFacturasProgramadas' });
        },
        selectedChanged(e){
            this.selectedRowIndex = e.component.getRowIndexByKey(e.selectedRowKeys[0]);
        },
        onRowClick(e){
            this.datosgen = e.data;
        },
        onRowDblClick(event){
            if(!this.permisos.update) return;
            this.datosgen = event.data;
            this.openEditar();
        },
        onRowExpanding(e){
            this.gridFacturacion.collapseAll(-1);
            this.gridFacturacion.getDataByKeys([e.key]).then(r => {
                this.datosgen = r[0];
            });
        },
        inicializarIntervalosFactura(){
            let nuevo = this.crearIntervaloBase(1);
            this.intervalosFactura = [nuevo];
            this.intervaloActivoId = nuevo.id;
            this.cargarEstadoVisibleDesdeIntervalo();
        },
        crearIntervaloBase(index = 1){
            return {
                id: `tmp_${Date.now()}_${index}`,
                nombre: `Inter ${index}`,
                tipoPeriodicidad: 'S',
                s01226instruccionGeneracion: '',
                reglaCadaPeriodos: 1,
                reglaIntervaloDias: 1,
                reglaFechaInicioUi: null,
                horaProgramadaUi: this.buildHoraDate('09:00'),
                mesesConfig: {}
            };
        },
        buildHoraDate(hora){
            let [hh, mm] = String(hora || '09:00').split(':');
            let fecha = new Date();
            fecha.setHours(Number(hh || 9), Number(mm || 0), 0, 0);
            return fecha;
        },
        formatoFechaCorta(value){
            if(!value) return '';
            let fecha = new Date(value);
            if(Number.isNaN(fecha.getTime())) return '';
            let anio = fecha.getFullYear();
            let mes = String(fecha.getMonth() + 1).padStart(2,'0');
            let dia = String(fecha.getDate()).padStart(2,'0');
            return `${anio}-${mes}-${dia}`;
        },
        formatoHoraCorta(value){
            if(!value) return '';
            let fecha = new Date(value);
            if(Number.isNaN(fecha.getTime())) return '';
            return fecha.toLocaleTimeString('es-MX', { hour: 'numeric', minute: '2-digit' });
        },
        formatoHoraMilitar(value){
            if(!value) return '';
            let fecha = new Date(value);
            if(Number.isNaN(fecha.getTime())) return '';
            return `${String(fecha.getHours()).padStart(2,'0')}:${String(fecha.getMinutes()).padStart(2,'0')}`;
        },
        cargarEstadoVisibleDesdeIntervalo(){
            let item = this.intervaloActivo;
            if(!item) return;
            this.tipoPeriodicidad = item.tipoPeriodicidad;
            this.reglaCadaPeriodos = Number(item.reglaCadaPeriodos || 1);
            this.reglaIntervaloDias = Number(item.reglaIntervaloDias || 1);
            this.reglaFechaInicioUi = item.reglaFechaInicioUi || null;
            this.datosgen.s01226intervaloGeneracion = item.tipoPeriodicidad;
            this.datosgen.s01226instruccionGeneracion = item.s01226instruccionGeneracion || '';
            this.datosgen.horaProgramadaUi = item.horaProgramadaUi || this.buildHoraDate('09:00');
            if(this.tipoPeriodicidad === 'M'){
                let activos = this.mesesSeleccionadosActivos;
                if(activos.length && !activos.includes(Number(this.mesActivoCalendario))){
                    this.mesActivoCalendario = activos[0];
                }
            }
        },
        guardarEstadoVisibleEnIntervalo(){
            let item = this.intervaloActivo;
            if(!item) return;
            item.tipoPeriodicidad = this.tipoPeriodicidad;
            item.s01226instruccionGeneracion = this.datosgen.s01226instruccionGeneracion || '';
            item.reglaCadaPeriodos = Number(this.reglaCadaPeriodos || 1);
            item.reglaIntervaloDias = Number(this.reglaIntervaloDias || 1);
            item.reglaFechaInicioUi = this.reglaFechaInicioUi || null;
            item.horaProgramadaUi = this.datosgen.horaProgramadaUi || null;
        },
        getTextoPillIntervalo(item){
            if(!item) return '';
            if(item.tipoPeriodicidad === 'X') return `Cada ${item.reglaIntervaloDias || 1} día(s)`;
            if(item.tipoPeriodicidad === 'M') return this.getResumenMesesYDias(item, true);
            return this.getTextoInstruccion(item.tipoPeriodicidad, item.s01226instruccionGeneracion) || 'Sin regla';
        },
        getTextoInstruccion(intervalo, instruccion){
            let arrVars = String(instruccion || '').split(',').map(item => item.trim()).filter(Boolean);
            if(!arrVars.length) return '';
            if(intervalo === 'S'){
                let dias = { '0':'lunes','1':'martes','2':'miércoles','3':'jueves','4':'viernes','5':'sábado','6':'domingo' };
                return arrVars.map(item => dias[item] || item).join(', ');
            }
            if(intervalo === 'Q'){
                let periodos = { '1':'día 1 del mes', '15':'día 15 del mes' };
                return arrVars.map(item => periodos[item] || `día ${item}`).join(', ');
            }
            return arrVars.join(', ');
        },
        getResumenMesesYDias(item, corto = false){
            let meses = Object.keys(item.mesesConfig || {}).map(v => Number(v)).sort((a,b)=>a-b);
            if(!meses.length) return corto ? 'Sin meses' : 'sin meses seleccionados';
            let partes = meses.map(m => {
                let dias = ((item.mesesConfig[m] || {}).dias || []).slice().sort((a,b)=>a-b);
                let mes = this.mesesCatalogo.find(x => x.value === Number(m));
                let nombreMes = corto ? mes.short : mes.text.toLowerCase();
                return `${nombreMes}: ${dias.join(', ') || 'sin días'}`;
            });
            return partes.join(corto ? ' | ' : ' · ');
        },
        agregarIntervaloFactura(){
            this.guardarEstadoVisibleEnIntervalo();
            let index = this.intervalosFactura.length + 1;
            let nuevo = this.crearIntervaloBase(index);
            this.intervalosFactura.push(nuevo);
            this.intervaloActivoId = nuevo.id;
            this.cargarEstadoVisibleDesdeIntervalo();
        },
        seleccionarIntervaloFacturaById(id){
            this.guardarEstadoVisibleEnIntervalo();
            this.intervaloActivoId = id;
            this.cargarEstadoVisibleDesdeIntervalo();
        },
        eliminarIntervaloFactura(id){
            if(this.intervalosFactura.length <= 1) return;
            this.guardarEstadoVisibleEnIntervalo();
            let idx = this.intervalosFactura.findIndex(item => String(item.id) === String(id));
            if(idx < 0) return;
            this.intervalosFactura.splice(idx, 1);
            this.intervalosFactura.forEach((item, index) => { item.nombre = `Inter ${index + 1}`; });
            let siguiente = this.intervalosFactura[idx] || this.intervalosFactura[idx - 1] || this.intervalosFactura[0];
            this.intervaloActivoId = siguiente.id;
            this.cargarEstadoVisibleDesdeIntervalo();
        },
        setFrecuenciaVisual(value){
            this.tipoPeriodicidad = value;
            this.datosgen.s01226intervaloGeneracion = value;
            if(value !== 'M'){
                this.datosgen.s01226instruccionGeneracion = '';
            }
            this.guardarEstadoVisibleEnIntervalo();
        },
        isSchedulerOptionSelected(value){
            let current = String(this.datosgen.s01226instruccionGeneracion || '').split(',').map(item => item.trim()).filter(Boolean);
            return current.includes(String(value));
        },
        toggleSchedulerOption(value){
            let current = String(this.datosgen.s01226instruccionGeneracion || '').split(',').map(item => item.trim()).filter(Boolean);
            let target = String(value);
            if(current.includes(target)){
                current = current.filter(item => item !== target);
            } else {
                current.push(target);
            }
            current = current.sort((a,b) => Number(a) - Number(b));
            this.datosgen.s01226instruccionGeneracion = current.join(',');
            this.guardarEstadoVisibleEnIntervalo();
        },
        seleccionarQuincenaRapida(values){
            this.datosgen.s01226instruccionGeneracion = values.join(',');
            this.guardarEstadoVisibleEnIntervalo();
        },
        decrementarIntervaloDias(){
            this.reglaIntervaloDias = Math.max(1, Number(this.reglaIntervaloDias || 1) - 1);
            this.guardarEstadoVisibleEnIntervalo();
        },
        incrementarIntervaloDias(){
            this.reglaIntervaloDias = Math.min(365, Number(this.reglaIntervaloDias || 1) + 1);
            this.guardarEstadoVisibleEnIntervalo();
        },
        seleccionarHoraRapida(hora){
            this.datosgen.horaProgramadaUi = this.buildHoraDate(hora);
            this.guardarEstadoVisibleEnIntervalo();
        },
        updateReglaFechaInicio(e){
            this.reglaFechaInicioUi = e.value;
            this.guardarEstadoVisibleEnIntervalo();
        },
        getDiasDisponiblesPorMes(mes){
            let anio = new Date().getFullYear();
            let ultimoDia = new Date(anio, Number(mes), 0).getDate();
            return Array.from({ length: ultimoDia }, (_, idx) => idx + 1);
        },
        isMesSeleccionado(mes){
            let item = this.intervaloActivo;
            if(!item || item.tipoPeriodicidad !== 'M') return false;
            return !!(item.mesesConfig || {})[mes];
        },
        toggleMes(mes){
            let item = this.intervaloActivo;
            if(!item) return;
            if(!item.mesesConfig) item.mesesConfig = {};
            if(item.mesesConfig[mes]){
                delete item.mesesConfig[mes];
                let activos = Object.keys(item.mesesConfig).map(v => Number(v)).sort((a,b)=>a-b);
                if(activos.length){
                    this.mesActivoCalendario = activos[0];
                }
            } else {
                item.mesesConfig[mes] = { dias: [] };
                this.mesActivoCalendario = Number(mes);
            }
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        seleccionarTodosMeses(){
            let item = this.intervaloActivo;
            if(!item) return;
            let nuevo = {};
            this.mesesCatalogo.forEach(m => {
                nuevo[m.value] = item.mesesConfig && item.mesesConfig[m.value]
                    ? item.mesesConfig[m.value]
                    : { dias: [] };
            });
            item.mesesConfig = nuevo;
            this.mesActivoCalendario = 1;
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        limpiarMeses(){
            let item = this.intervaloActivo;
            if(!item) return;
            item.mesesConfig = {};
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        mesActivoAnterior(){
            let activos = this.mesesSeleccionadosActivos;
            if(!activos.length) return;
            let idx = activos.indexOf(Number(this.mesActivoCalendario));
            if(idx <= 0){
                this.mesActivoCalendario = activos[activos.length - 1];
            } else {
                this.mesActivoCalendario = activos[idx - 1];
            }
        },
        mesActivoSiguiente(){
            let activos = this.mesesSeleccionadosActivos;
            if(!activos.length) return;
            let idx = activos.indexOf(Number(this.mesActivoCalendario));
            if(idx === -1 || idx >= activos.length - 1){
                this.mesActivoCalendario = activos[0];
            } else {
                this.mesActivoCalendario = activos[idx + 1];
            }
        },
        isDiaSeleccionadoEnMesActivo(dia){
            let item = this.intervaloActivo;
            if(!item || !item.mesesConfig || !item.mesesConfig[this.mesActivoCalendario]) return false;
            return (item.mesesConfig[this.mesActivoCalendario].dias || []).includes(Number(dia));
        },
        toggleDiaMesActivo(dia){
            let item = this.intervaloActivo;
            if(!item || !item.mesesConfig || !item.mesesConfig[this.mesActivoCalendario]) return;
            let dias = item.mesesConfig[this.mesActivoCalendario].dias || [];
            let n = Number(dia);
            if(dias.includes(n)){
                dias = dias.filter(v => v !== n);
            } else {
                dias.push(n);
            }
            dias.sort((a,b) => a-b);
            item.mesesConfig[this.mesActivoCalendario].dias = dias;
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        seleccionarDiaMesActivo(dia){
            let item = this.intervaloActivo;
            if(!item || !item.mesesConfig || !item.mesesConfig[this.mesActivoCalendario]) return;
            let maxDia = this.getDiasDisponiblesPorMes(this.mesActivoCalendario).slice(-1)[0];
            if(Number(dia) > Number(maxDia)) return;
            item.mesesConfig[this.mesActivoCalendario].dias = [Number(dia)];
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        seleccionarUltimoDiaMesActivo(){
            let maxDia = this.getDiasDisponiblesPorMes(this.mesActivoCalendario).slice(-1)[0];
            this.seleccionarDiaMesActivo(maxDia);
        },
        limpiarDiasMesActivo(){
            let item = this.intervaloActivo;
            if(!item || !item.mesesConfig || !item.mesesConfig[this.mesActivoCalendario]) return;
            item.mesesConfig[this.mesActivoCalendario].dias = [];
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        aplicarDiaATodosLosMeses(dia){
            let item = this.intervaloActivo;
            if(!item) return;
            if(!item.mesesConfig) item.mesesConfig = {};
            Object.keys(item.mesesConfig).forEach(m => {
                let maxDia = this.getDiasDisponiblesPorMes(Number(m)).slice(-1)[0];
                if(Number(dia) <= Number(maxDia)){
                    item.mesesConfig[m].dias = [Number(dia)];
                } else {
                    item.mesesConfig[m].dias = [];
                }
            });
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        aplicarUltimoDiaATodosLosMeses(){
            let item = this.intervaloActivo;
            if(!item) return;
            if(!item.mesesConfig) item.mesesConfig = {};
            Object.keys(item.mesesConfig).forEach(m => {
                let maxDia = this.getDiasDisponiblesPorMes(Number(m)).slice(-1)[0];
                item.mesesConfig[m].dias = [Number(maxDia)];
            });
            this.serializarMesesConfig();
            this.guardarEstadoVisibleEnIntervalo();
        },
        serializarMesesConfig(){
            let item = this.intervaloActivo;
            if(!item) return;
            item.s01226instruccionGeneracion = JSON.stringify(item.mesesConfig || {});
            this.datosgen.s01226instruccionGeneracion = item.s01226instruccionGeneracion;
        },
        deserializarMesesConfig(raw){
            try {
                let parsed = JSON.parse(raw || '{}');
                return parsed && typeof parsed === 'object' ? parsed : {};
            } catch (error) {
                return {};
            }
        },
        getSchedulerMeta(intervalo){
            let origen = intervalo || this.intervaloActivo || {};
            return {
                version: 3,
                mode: origen.tipoPeriodicidad,
                days: origen.tipoPeriodicidad === 'M'
                    ? null
                    : String(origen.s01226instruccionGeneracion || '').split(',').map(v => v.trim()).filter(Boolean),
                mesesConfig: origen.tipoPeriodicidad === 'M' ? (origen.mesesConfig || {}) : null,
                interval_step: origen.tipoPeriodicidad === 'X' ? null : Number(origen.reglaCadaPeriodos || 1),
                every: origen.tipoPeriodicidad === 'X' ? Number(origen.reglaIntervaloDias || 1) : null,
                start_date: origen.reglaFechaInicioUi ? this.formatoFechaCorta(origen.reglaFechaInicioUi) : null,
                factura_base_id: this.datosgen.i01220idFacturaBase || null,
                intervalos: this.intervalosFactura.map(item => ({
                    nombre: item.nombre,
                    mode: item.tipoPeriodicidad,
                    days: item.tipoPeriodicidad === 'M' ? null : String(item.s01226instruccionGeneracion || '').split(',').map(v => v.trim()).filter(Boolean),
                    mesesConfig: item.tipoPeriodicidad === 'M' ? (item.mesesConfig || {}) : null,
                    interval_step: item.tipoPeriodicidad === 'X' ? null : Number(item.reglaCadaPeriodos || 1),
                    every: item.tipoPeriodicidad === 'X' ? Number(item.reglaIntervaloDias || 1) : null,
                    start_date: item.reglaFechaInicioUi ? this.formatoFechaCorta(item.reglaFechaInicioUi) : null,
                    hour: item.horaProgramadaUi ? this.formatoHoraMilitar(item.horaProgramadaUi) : null
                }))
            };
        },
        mergeComentariosConMeta(comentarioVisible, intervalo){
            let meta = this.getSchedulerMeta(intervalo);
            let encoded = encodeURIComponent(JSON.stringify(meta));
            let comentario = comentarioVisible || '';
            return `[[FPMETA]]${encoded}\n${comentario}`;
        },
        parseComentariosConMeta(raw){
            let texto = String(raw || '');
            let prefix = '[[FPMETA]]';
            if(texto.indexOf(prefix) !== 0){
                return { comentario: texto, meta: null };
            }
            let lineas = texto.split(/\r\n|\n|\r/);
            let primera = lineas.shift() || '';
            let encoded = primera.replace(prefix, '');
            let meta = null;
            try { meta = JSON.parse(decodeURIComponent(encoded)); } catch(error){ meta = null; }
            return { comentario: lineas.join('\n'), meta };
        },
        normalizarHoraProgramada(value){
            let fecha = value ? new Date(value) : new Date();
            if(Number.isNaN(fecha.getTime())) fecha = new Date();
            return { hora: fecha.getHours(), minuto: fecha.getMinutes() };
        },
        facturacionGrabar(){
            this.guardarEstadoVisibleEnIntervalo();
            if(!this.validateProgramacion()) return;
            if(this.formEdit.validate().isValid){
                this.gridFacturacion.saveEditData();
            } else {
                let brokenRules = [];
                this.formEdit.validate().brokenRules.forEach(rule => brokenRules.push(rule.message));
                this.error('Favor de completar los campos, ' + brokenRules.join(','), 10000);
            }
        },
        facturacionCerrar(){
            this.gridFacturacion.cancelEditData();
        },
        validateProgramacion(){
            if(!this.datosgen.i01220idFacturaBase){
                this.error('Selecciona la factura base.', 10000);
                return false;
            }
            if(!this.intervalosFactura.length){
                this.error('Agrega al menos un intervalo.', 10000);
                return false;
            }
            for(let i = 0; i < this.intervalosFactura.length; i++){
                let item = this.intervalosFactura[i];
                if(!item.tipoPeriodicidad){
                    this.error(`Inter ${i + 1}: selecciona una frecuencia.`, 10000);
                    return false;
                }
                if(item.tipoPeriodicidad === 'M'){
                    let meses = Object.keys(item.mesesConfig || {});
                    if(!meses.length){
                        this.error(`Inter ${i + 1}: selecciona al menos un mes.`, 10000);
                        return false;
                    }
                    let conDias = meses.some(m => ((item.mesesConfig[m] || {}).dias || []).length > 0);
                    if(!conDias){
                        this.error(`Inter ${i + 1}: selecciona al menos un día en los meses elegidos.`, 10000);
                        return false;
                    }
                } else if(item.tipoPeriodicidad === 'X'){
                    if(!Number(item.reglaIntervaloDias || 0)){
                        this.error(`Inter ${i + 1}: indica cada cuántos días debe repetirse.`, 10000);
                        return false;
                    }
                } else if(!item.s01226instruccionGeneracion){
                    this.error(`Inter ${i + 1}: selecciona al menos una regla.`, 10000);
                    return false;
                }
                if(!item.horaProgramadaUi){
                    this.error(`Inter ${i + 1}: selecciona la hora de generación.`, 10000);
                    return false;
                }
            }
            return true;
        },
        grabar(e){
            this.guardarEstadoVisibleEnIntervalo();
            let item = this.intervalosFactura[0] || this.crearIntervaloBase(1);
            let instruccionCompacta = item.tipoPeriodicidad === 'M'
                ? 'META'
                : item.tipoPeriodicidad === 'X'
                    ? String(item.reglaIntervaloDias || 1)
                    : ((item.s01226instruccionGeneracion && String(item.s01226instruccionGeneracion).length <= 20)
                        ? item.s01226instruccionGeneracion
                        : 'META');

            let comentarioVisible = e.data
                ? e.data.s01226comentarios
                : (e.newData && e.newData.s01226comentarios !== undefined
                    ? e.newData.s01226comentarios
                    : this.datosgen.s01226comentarios);

            let comentarioConMeta = this.mergeComentariosConMeta(comentarioVisible, item);
            let data = {};
            let horaProgramada = this.normalizarHoraProgramada(item.horaProgramadaUi || this.datosgen.horaProgramadaUi);

            if(this.isNuevo){
                if(e.data){
                    data.i0011idEmpresa = this.$root.$children[0].empresa;
                    data.i0012idSucursal = window.sucursal;
                    data.i01220idFacturaBase = e.data.i01220idFacturaBase;
                    data.s01226referencia = e.data.s01226referencia;
                    data.s01226comentarios = comentarioConMeta;
                    data.s01226intervaloGeneracion = item.tipoPeriodicidad;
                    data.s01226instruccionGeneracion = instruccionCompacta;
                    data.i01226horaGeneracion = horaProgramada.hora;
                    data.i01226minutoGeneracion = horaProgramada.minuto;
                    data.i0017idUsuarioCrea = this.$root.$children[0].usuario.i0017idUsuario;

                    this.xquery('insert', 'sac01226', data).then(() => {
                        this.gridFacturacion.refresh();
                        this.facturacionCerrar();
                    });
                }
            } else {
                data = e.newData;
                delete data.horaProgramadaUi;
                data.i0011idEmpresa = this.$root.$children[0].empresa;
                data.i0012idSucursal = window.sucursal;
                data.s01226intervaloGeneracion = item.tipoPeriodicidad;
                data.s01226instruccionGeneracion = instruccionCompacta;
                data.i01226horaGeneracion = horaProgramada.hora;
                data.i01226minutoGeneracion = horaProgramada.minuto;
                data.s01226comentarios = comentarioConMeta;
                data.i0017idUsuarioActualiza = this.$root.$children[0].usuario.i0017idUsuario;
                data.d01226actualiza = this.getDate('datetime');

                this.xquery('update', 'sac01226', data, `i01226idFacturaProgramada=${this.datosgen.i01226idFacturaProgramada}`).then(() => {
                    this.gridFacturacion.refresh();
                    this.facturacionCerrar();
                });
            }

            e.cancel = true;
        },
        calculateInstrucciones(e){
            let comentarios = this.parseComentariosConMeta(e.s01226comentarios);
            if(comentarios.meta && comentarios.meta.mode === 'M' && comentarios.meta.mesesConfig){
                let item = { tipoPeriodicidad: 'M', mesesConfig: comentarios.meta.mesesConfig };
                return this.getResumenMesesYDias(item, true);
            }
            if(comentarios.meta && comentarios.meta.mode === 'X'){
                let base = `cada ${comentarios.meta.every || 1} día(s)`;
                let inicio = comentarios.meta.start_date ? ` desde ${comentarios.meta.start_date}` : '';
                return `${base}${inicio}`;
            }
            if(comentarios.meta){
                return this.getTextoInstruccion(
                    comentarios.meta.mode || e.s01226intervaloGeneracion,
                    Array.isArray(comentarios.meta.days) ? comentarios.meta.days.join(',') : e.s01226instruccionGeneracion
                );
            }
            return e.s01226intervaloGeneracion === 'X'
                ? `cada ${e.s01226instruccionGeneracion || 1} día(s)`
                : this.getTextoInstruccion(e.s01226intervaloGeneracion, e.s01226instruccionGeneracion);
        },
        calculateComentariosVisibles(e){
            return this.parseComentariosConMeta(e.s01226comentarios).comentario;
        },
        calculateHoras(e){
            let hora = String(e.i01226horaGeneracion).padStart(2, '0');
            let minuto = String(e.i01226minutoGeneracion || 0).padStart(2, '0');
            return `${hora}:${minuto}`;
        },
        changeConfiguracionClientePillManual(id){
            this.datosgen.configuracionClienteActiva = id;
            this.cargarProgramacionClienteSeleccionada();
        },
        cargarProgramacionClienteSeleccionada(){
            if(!this.datosgen.configuracionClienteActiva){
                this.info('Selecciona una configuración del cliente para cargarla.', 5000);
                return;
            }
            let seleccionada = this.clienteProgramacionesRaw.find(item => String(item.i01226idFacturaProgramada) === String(this.datosgen.configuracionClienteActiva));
            if(!seleccionada){
                this.info('No se encontró la configuración seleccionada.', 5000);
                return;
            }

            let comentarios = this.parseComentariosConMeta(seleccionada.s01226comentarios);
            this.datosgen.s01226comentarios = comentarios.comentario;
            this.datosgen.s01226referencia = seleccionada.s01226referencia || this.datosgen.s01226referencia;
            this.datosgen.i01220idFacturaBase = (comentarios.meta && comentarios.meta.factura_base_id) || seleccionada.i01220idFacturaBase || this.datosgen.i01220idFacturaBase;

            let intervalo = this.crearIntervaloBase(1);
            intervalo.tipoPeriodicidad = (comentarios.meta && comentarios.meta.mode) || seleccionada.s01226intervaloGeneracion || 'S';
            intervalo.s01226instruccionGeneracion = comentarios.meta && Array.isArray(comentarios.meta.days)
                ? comentarios.meta.days.join(',')
                : (seleccionada.s01226instruccionGeneracion || '');
            intervalo.reglaIntervaloDias = Number((comentarios.meta && comentarios.meta.every) || 1);
            intervalo.reglaCadaPeriodos = Number((comentarios.meta && comentarios.meta.interval_step) || 1);
            intervalo.reglaFechaInicioUi = comentarios.meta ? (comentarios.meta.start_date || null) : null;
            intervalo.horaProgramadaUi = this.buildHoraDate((comentarios.meta && comentarios.meta.intervalos && comentarios.meta.intervalos[0] && comentarios.meta.intervalos[0].hour) || this.calculateHoras(seleccionada));
            intervalo.mesesConfig = comentarios.meta && comentarios.meta.mesesConfig ? comentarios.meta.mesesConfig : {};

            this.intervalosFactura = [intervalo];
            this.intervaloActivoId = intervalo.id;
            let meses = Object.keys(intervalo.mesesConfig || {}).map(v => Number(v)).sort((a,b)=>a-b);
            if(meses.length){
                this.mesActivoCalendario = meses[0];
            }
            this.cargarEstadoVisibleDesdeIntervalo();
        },
        openNuevo(){
            this.isNuevo = true;
            this.clienteProgramaciones = [];
            this.clienteProgramacionesRaw = [];
            this.datosgen = {
                configuracionClienteActiva: null,
                i01220idFacturaBase: null,
                s01226referencia: '',
                s01226comentarios: '',
                horaProgramadaUi: this.buildHoraDate('09:00')
            };
            this.intervalosFactura = [this.crearIntervaloBase(1)];
            this.intervaloActivoId = this.intervalosFactura[0].id;
            this.cargarEstadoVisibleDesdeIntervalo();
            this.gridFacturacion.addRow();
        },
        openEditar(){
            if(this.selectedRowIndex !== -1){
                this.isNuevo = false;
                this.gridFacturacion.editRow(this.selectedRowIndex);
            } else {
                this.info('Favor de seleccionar un registro', 5000);
            }
        },
        editStart(event){
            let comentarios = this.parseComentariosConMeta(event.data.s01226comentarios);
            this.datosgen = Object.assign({}, event.data, {
                s01226comentarios: comentarios.comentario,
                i01220idFacturaBase: (comentarios.meta && comentarios.meta.factura_base_id) || event.data.i01220idFacturaBase || null
            });

            let intervalo = this.crearIntervaloBase(1);
            intervalo.tipoPeriodicidad = (comentarios.meta && comentarios.meta.mode) || event.data.s01226intervaloGeneracion || 'S';
            intervalo.s01226instruccionGeneracion = comentarios.meta && Array.isArray(comentarios.meta.days)
                ? comentarios.meta.days.join(',')
                : (event.data.s01226instruccionGeneracion || '');
            intervalo.reglaIntervaloDias = Number((comentarios.meta && comentarios.meta.every) || 1);
            intervalo.reglaCadaPeriodos = Number((comentarios.meta && comentarios.meta.interval_step) || 1);
            intervalo.reglaFechaInicioUi = comentarios.meta ? (comentarios.meta.start_date || null) : null;
            intervalo.horaProgramadaUi = this.buildHoraDate((comentarios.meta && comentarios.meta.intervalos && comentarios.meta.intervalos[0] && comentarios.meta.intervalos[0].hour) || this.calculateHoras(event.data));
            intervalo.mesesConfig = comentarios.meta && comentarios.meta.mesesConfig ? comentarios.meta.mesesConfig : {};

            this.intervalosFactura = [intervalo];
            this.intervaloActivoId = intervalo.id;
            let meses = Object.keys(intervalo.mesesConfig || {}).map(v => Number(v)).sort((a,b)=>a-b);
            if(meses.length){
                this.mesActivoCalendario = meses[0];
            }
            this.cargarEstadoVisibleDesdeIntervalo();
        },
        borrarProgramacion(){
            if(this.selectedRowIndex !== -1){
                this.confirm('Desea desactivar los registros?', () => {
                    let data = [];
                    this.gridFacturacion.getSelectedRowsData().forEach(item => data.push(item.i01226idFacturaProgramada));
                    this.xquery('update', 'sac01226', { i01226activo: 0 }, ' i01226idFacturaProgramada IN(' + data.join(',') + ')').then(() => {
                        this.gridFacturacion.clearSelection();
                        this.gridFacturacion.refresh();
                    });
                }, () => {});
            } else {
                this.info('Favor de seleccionar un registro', 5000);
            }
        },
        activarProgramacion(){
            if(this.selectedRowIndex !== -1){
                this.confirm('Desea activar los registros?', () => {
                    let data = [];
                    this.gridFacturacion.getSelectedRowsData().forEach(item => data.push(item.i01226idFacturaProgramada));
                    this.xquery('update', 'sac01226', { i01226activo: 1 }, ' i01226idFacturaProgramada IN(' + data.join(',') + ')').then(() => {
                        this.gridFacturacion.clearSelection();
                        this.gridFacturacion.refresh();
                    });
                }, () => {});
            } else {
                this.info('Favor de seleccionar un registro', 5000);
            }
        }
    }
};
</script>

<style scoped>
.programacion-builder{
    margin-top: 10px;
    padding: 14px;
    border: 1px solid rgba(255,255,255,0.08);
    border-radius: 16px;
    background: radial-gradient(circle at top left, rgba(77, 208, 225, 0.10), transparent 32%), linear-gradient(180deg, rgba(255,255,255,0.04), rgba(255,255,255,0.015));
    box-shadow: inset 0 1px 0 rgba(255,255,255,0.04);
}

.factura-programada-form{
    padding-top: 6px;
}

.factura-programada-form :deep(.fp-card){
    padding: 14px;
    border-radius: 16px;
    background: rgba(255,255,255,0.02);
    border: 1px solid rgba(255,255,255,0.07);
    min-height: 100%;
}

.factura-programada-form :deep(.dx-field-item-group-caption){
    font-size: 14px;
    font-weight: 700;
    color: #f3f3f3;
    margin-bottom: 10px;
}

.fp-builder-v3{
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.fp-section{
    padding: 4px 0 16px 0;
    border-bottom: 1px solid rgba(120,120,140,0.15);
}

.fp-label{
    font-size: 14px;
    font-weight: 700;
    margin-bottom: 6px;
    color: #f3f3f3;
}

.fp-help{
    font-size: 12px;
    opacity: .82;
    line-height: 1.45;
    color: #d4d7df;
}

.mt-8{ margin-top: 8px; }

.fp-pills-wrap{
    display:flex;
    flex-wrap:wrap;
    gap:10px;
    margin-top:12px;
}

.fp-mini-pill{
    border: 1px solid rgba(255,255,255,0.14);
    background: rgba(255,255,255,0.03);
    color: #f3f3f3;
    border-radius: 999px;
    min-height: 38px;
    padding: 0 14px;
    font-weight: 600;
}

.fp-mini-pill.active{
    border-color:#c784ff;
    background: rgba(176,80,255,0.18);
}

.fp-mini-pill-editable{
    display:inline-flex;
    align-items:center;
    gap:8px;
}

.fp-mini-pill-hint{
    font-size:11px;
    opacity:.7;
}

.fp-pill-remove{
    width: 18px;
    height: 18px;
    display:inline-flex;
    align-items:center;
    justify-content:center;
    border-radius: 999px;
    background: rgba(255,255,255,0.12);
    font-size: 13px;
}

.fp-add-row{
    margin-top: 12px;
}

.fp-outline-btn{
    border-radius: 12px;
    border: 1px solid #c784ff;
    background: transparent;
    color: #d88eff;
    padding: 10px 14px;
    font-weight: 700;
}

.fp-step-title{
    display:flex;
    align-items:center;
    gap:10px;
    margin-bottom:14px;
    font-size:15px;
    font-weight:700;
    color:#f3f3f3;
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
    font-weight:700;
}

.fp-frequency-grid{
    display:grid;
    grid-template-columns:repeat(2,minmax(0,1fr));
    gap:14px;
}

.fp-frequency-card{
    min-height:86px;
    padding:18px;
    border-radius:16px;
    border:1px solid rgba(255,255,255,.12);
    background: rgba(255,255,255,.03);
    text-align:left;
    transition:.18s ease;
    color:#fff;
}

.fp-frequency-card strong{
    display:block;
    font-size:15px;
    margin-bottom:6px;
    color:#fff;
}

.fp-frequency-card span{
    font-size:13px;
    color:#d9dbe1;
}

.fp-frequency-card.active{
    border:2px solid #b050ff;
    background:rgba(176,80,255,0.12);
    box-shadow:0 0 0 3px rgba(176,80,255,0.08);
}

.fp-chip-grid{
    display:grid;
    gap:12px;
}

.fp-chip-grid.days{
    grid-template-columns:repeat(7,minmax(0,1fr));
}

.fp-chip-grid.quincena{
    grid-template-columns:repeat(2,minmax(0,1fr));
}

.fp-chip-grid.month-days-calendar{
    grid-template-columns:repeat(7,minmax(0,1fr));
}

.fp-chip-card,
.fp-day-card,
.fp-month-card{
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
    gap:4px;
}

.fp-chip-card small{
    font-size: 11px;
    opacity: .75;
}

.fp-chip-card.active,
.fp-day-card.active,
.fp-month-card.active{
    border:2px solid #b050ff;
    background:rgba(176,80,255,0.12);
    color:#f3d8ff;
}

.fp-quick-row{
    display:flex;
    gap:10px;
    flex-wrap:wrap;
    margin-bottom:14px;
}

.fp-quick-row.compact{
    margin-bottom: 0;
}

.fp-quick-chip{
    padding:8px 12px;
    border-radius:12px;
    border:1px solid #f0b84b;
    background:rgba(240,184,75,.08);
    color:#ffd275;
    font-weight:700;
}

.fp-inline-grid{
    display:grid;
    grid-template-columns:repeat(2,minmax(0,1fr));
    gap:18px;
}

.fp-inline-grid.one-col{
    grid-template-columns:1fr;
}

.fp-number-row,
.fp-stepper-wrap{
    display:flex;
    align-items:center;
    gap:10px;
}

.fp-stepper-btn{
    width:38px;
    height:38px;
    border:1px solid rgba(255,255,255,.14);
    background:rgba(255,255,255,.03);
    color:#fff;
    border-radius:10px;
    font-size:22px;
    line-height:1;
}

.fp-stepper-value{
    min-width:140px;
    text-align:center;
    padding:10px 14px;
    border-radius:12px;
    background:rgba(255,255,255,.03);
    border:1px solid rgba(255,255,255,.14);
    color:#fff;
    font-weight:700;
}

.fp-stepper-value.month-viewer{
    min-width: 220px;
}

.fp-hours-wrap{
    display:flex;
    gap:10px;
    flex-wrap:wrap;
}

.fp-hour-chip{
    min-height:42px;
    padding:0 14px;
    border-radius:12px;
    border:1px solid #7bb2ff;
    background:rgba(41,121,255,.08);
    color:#b9d4ff;
    font-weight:700;
}

.fp-hour-chip.active{
    background:rgba(41,121,255,.18);
    border-color:#7bb2ff;
    color:#fff;
}

.fp-months-grid{
    display:grid;
    grid-template-columns: repeat(4, minmax(0,1fr));
    gap: 12px;
    margin-bottom: 16px;
}

.fp-month-card strong{
    font-size: 16px;
}

.fp-month-card span{
    font-size: 12px;
    opacity: .75;
}

.fp-month-navigator{
    margin-bottom: 16px;
}

.fp-month-calendar-card{
    padding: 16px;
    border-radius: 16px;
    background: rgba(255,255,255,.03);
    border: 1px solid rgba(255,255,255,.12);
}

.fp-calendar-header{
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 14px;
}

.fp-calendar-header strong{
    display: block;
    color: #fff;
    font-size: 15px;
}

.fp-calendar-header span{
    display: block;
    font-size: 12px;
    opacity: .75;
    color: #d4d7df;
}

.fp-empty-note{
    padding: 16px;
    border-radius: 14px;
    background: rgba(255,255,255,.03);
    border: 1px dashed rgba(255,255,255,.16);
    color: #d4d7df;
}

.fp-preview-card{
    padding:20px;
    border-radius:18px;
    background:linear-gradient(135deg, #a100ff, #7d14ff);
    color:#fff;
    box-shadow:0 12px 28px rgba(125,20,255,0.18);
}

.fp-preview-title{
    font-size:16px;
    font-weight:800;
    margin-bottom:8px;
}

.fp-preview-text{
    font-size:14px;
    line-height:1.6;
}

.programacion-builder :deep(.dx-texteditor),
.programacion-builder :deep(.dx-dropdowneditor-button),
.programacion-builder :deep(.dx-datebox){
    border-radius: 12px;
}

.programacion-builder :deep(.dx-texteditor.dx-editor-outlined),
.programacion-builder :deep(.dx-texteditor){
    background: rgba(255,255,255,0.03);
    border: 1px solid rgba(255,255,255,0.08);
}

@media (max-width: 900px){
    .fp-frequency-grid,
    .fp-inline-grid,
    .fp-months-grid{
        grid-template-columns:1fr;
    }

    .fp-chip-grid.days,
    .fp-chip-grid.month-days-calendar{
        grid-template-columns:repeat(3,minmax(0,1fr));
    }

    .fp-chip-grid.quincena{
        grid-template-columns:1fr;
    }

    .fp-calendar-header{
        flex-direction: column;
        align-items: flex-start;
    }
}
</style>
