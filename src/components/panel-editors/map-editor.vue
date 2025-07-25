<template>
    <div class="flex flex-col mt-4">
        <label class="respected-standard-label text-left" for="mapTitle">{{ $t('editor.map.title') }}</label>
        <input class="respected-standard-input" type="text" id="mapTitle" v-model="panel.title" />

        <div>
            <div class="flex items-center gap-2 mt-2">
                <input
                    class="rounded-none cursor-pointer w-4 h-4"
                    type="checkbox"
                    id="timeSliderToggle"
                    @change="saveTimeSlider"
                    v-model="usingTimeSlider"
                />
                <label class="respected-standard-label" for="timeSliderToggle">{{
                    $t('editor.map.timeslider.enable')
                }}</label>
                <!-- Use invisible instead of v-if to reserve space for button (so checkbox doesn't move around) -->
                <button
                    :class="{ 'hidden md:invisible': !usingTimeSlider }"
                    @click="$vfm.open('time-slider-edit-modal')"
                    class="respected-standard-button respected-black-bg-button respected-thin-button block md:inline-block ml-0 text-sm"
                >
                    {{ $t('editor.map.timeslider.edit') }}
                </button>
            </div>

            <div class="flex flex-col w-full text-left mt-5 mb-1">
                <label class="respected-standard-label" for="rampMapCaption">
                    {{ $t('editor.image.label.caption') }}</label
                >
                <input
                    id="rampMapCaption"
                    class="respected-standard-input w-full lg:w-2/5"
                    type="text"
                    v-model="panel.caption"
                    :placeholder="$t('editor.caption.placeholder')"
                />
            </div>

            <div
                class="ramp-editor mt-5"
                ref="editor"
                style="width: 70vw; height: 80vh"
                @input="stateStore.overrideChangeState(true)"
            ></div>
        </div>
        <vue-final-modal
            modalId="time-slider-edit-modal"
            content-class="flex flex-col max-w-xl mx-4 p-4 bg-white border rounded-lg space-y-2"
            class="flex justify-center items-center"
        >
            <h2 slot="header" class="text-xl font-bold">{{ $t('editor.map.timeslider.edit') }}</h2>
            <time-slider-editor
                :config="timeSliderConf"
                :error="timeSliderError"
                @time-slider-changed="onTimeSliderInput"
            ></time-slider-editor>
            <div class="w-full flex justify-end">
                <button
                    class="respected-standard-button respected-black-bg-button"
                    :class="timeSliderError ? '' : 'bg-black text-white hover:bg-gray-800'"
                    :disabled="timeSliderError"
                    @click="saveTimeSlider"
                >
                    {{ $t('editor.done') }}
                </button>
            </div>
        </vue-final-modal>
    </div>
</template>

<script lang="ts">
import { useStateStore } from '@/stores/stateStore';
import { Options, Prop, Vue } from 'vue-property-decorator';
import { MapPanel, TimeSliderConfig } from '@/definitions';
import { applyTextAlign } from '@/utils/styleUtils';
import { VueFinalModal } from 'vue-final-modal';
import defaultConfig from '../../../ramp-default.json';
import TimeSliderEditorV from '../support/time-slider-editor.vue';
import { createInstance as createRampEditorInstance } from 'ramp-config-editor_editeur-config-pcar';
import 'ramp-config-editor_editeur-config-pcar/style.css';
import { useProductStore } from '@/stores/productStore';

@Options({
    components: {
        'time-slider-editor': TimeSliderEditorV,
        'vue-final-modal': VueFinalModal
    }
})
export default class MapEditorV extends Vue {
    @Prop() panel!: MapPanel;
    @Prop() lang!: string;
    @Prop({ default: false }) centerSlide!: boolean;
    @Prop({ default: false }) dynamicSelected!: boolean;

    stateStore = useStateStore();
    productStore = useProductStore();

    // config editor
    rampEditorApi: any = '';
    currLang = 'en';

    // For creating new files.
    newFileName = '';

    // TimeSlider
    usingTimeSlider = false;
    timeSliderError = false;
    timeSliderConf: TimeSliderConfig = { range: [], start: [], attribute: '' };
    status = 'default';
    strippedFileName = '';

    mounted(): void {
        this.currLang = (this.$route.params.lang as string) || 'en';
        this.usingTimeSlider = !!this.panel.timeSlider;
        this.status = this.panel.config !== '' ? 'default' : 'creating';
        this.strippedFileName = this.panel.config !== '' ? this.panel.config.split('/')[2].split('.')[0] : '';

        this.timeSliderConf = JSON.parse(
            JSON.stringify({
                range: this.panel.timeSlider?.range ?? [1000, new Date().getFullYear()],
                start: this.panel.timeSlider?.start ?? [1000, new Date().getFullYear()],
                attribute: this.panel.timeSlider?.attribute ?? ''
            })
        );
        window.addEventListener('ramp4-config-edited', this.onConfigEdit);
        this.validateTimeSlider();

        if (this.status === 'creating') {
            this.createNewConfig();
        }

        applyTextAlign(this.panel, this.centerSlide, this.dynamicSelected);
        this.openEditor();
    }

    beforeUnmount(): void {
        this.$emit('slide-edit', 'Kill map editor');
    }

    beforeDestroy(): void {
        window.removeEventListener('ramp4-config-edited', this.onConfigEdit);
    }

    createNewConfig(): void {
        // Update the path to the new file.
        const uuid = this.productStore.configFileStructure.uuid;
        this.panel.config = `${uuid}/ramp-config/${uuid}-map-${this.getNumberOfMaps() + 1}.json`;
        this.strippedFileName = this.panel.config.split('/')[2].split('.')[0];

        this.productStore.incrementSourceCount(this.panel.config);

        // Create the new map configuration file in the ZIP folder. Copies the `config-default.json` file from the `ramp-editor` folder and renames it.
        this.productStore.configFileStructure.rampConfig.file(
            `${this.strippedFileName}.json`,
            JSON.stringify(defaultConfig, null, 4)
        );

        // Display the normal edit page now.
        this.status = 'default';
    }

    openEditor(): void {
        if (this.panel.config === '') {
            return;
        }
        // Fetch the map configuration and load it into the editor.
        this.status = 'editing';

        if (this.panel.config) {
            // Check if the config file exists in the ZIP folder first.
            const assetSrc = `${this.panel.config.substring(this.panel.config.indexOf('/') + 1)}`;
            const configFile = this.productStore.configFileStructure.zip.file(assetSrc);

            if (configFile) {
                configFile.async('string').then((res: string) => {
                    const conf = JSON.parse(res);
                    this.rampEditorApi = createRampEditorInstance(this.$refs.editor, conf);
                    this.rampEditorApi.setLanguage(this.currLang);
                });
            } else {
                // If it does not exist in the ZIP folder, try and fetch from server.
                fetch(this.panel.config).then((data) => {
                    data.json().then((res) => {
                        let stringResponse = JSON.stringify(res);
                        const conf = JSON.parse(stringResponse);
                        this.rampEditorApi = createRampEditorInstance(this.$refs.editor, conf);
                        this.rampEditorApi.setLanguage(this.currLang);
                    });
                });
            }
        }
    }

    saveTimeSlider(): void {
        if (!this.timeSliderError || !this.usingTimeSlider) {
            this.panel.timeSlider = this.usingTimeSlider ? this.timeSliderConf : undefined;
        }
        this.$emit('slide-edit', 'Map editor time slider');
        this.$vfm.close('time-slider-edit-modal');
    }

    saveChanges(): void {
        // Add map config to ZIP file.
        this.productStore.configFileStructure.rampConfig.file(
            `${this.strippedFileName}.json`,
            JSON.stringify(this.rampEditorApi.getConfig(), null, 4)
        );
    }

    onConfigEdit(): void {
        this.$emit('slide-edit', 'Map editor');
    }

    onTimeSliderInput(property: 'range' | 'start' | 'attribute' | 'layers', index: number, value: string): void {
        if (property === 'layers') {
            if (!value || value === '') {
                delete this.timeSliderConf['layers'];
            } else {
                this.timeSliderConf['layers'] = value.split(',').map((layerId) => {
                    return layerId.trim();
                });
            }
        } else {
            property === 'attribute'
                ? (this.timeSliderConf[property] = value)
                : (this.timeSliderConf[property][index] = Number(value));
        }
        this.validateTimeSlider();
    }

    validateTimeSlider(): void {
        this.timeSliderError =
            this.timeSliderConf.range.some((val) => val < 0 || !Number.isInteger(val)) ||
            this.timeSliderConf.start.some((val) => val < 0 || !Number.isInteger(val)) ||
            this.timeSliderConf.range[1] < this.timeSliderConf.range[0] ||
            this.timeSliderConf.start[1] < this.timeSliderConf.start[0];
    }

    getNumberOfMaps(): number {
        let n = 0;
        this.productStore.configFileStructure.rampConfig.forEach((_f) => {
            n += 1;
        });
        return n;
    }
}
</script>

<style lang="scss" scoped>
label {
    text-align: left !important;
    width: fit-content !important;
}

select {
    border: 1px black solid;
    background: white;
    padding: 0.25rem 0.5rem;
}

.map-item {
    width: 300px;
    background: #eee;

    text-align: center;
    padding: 25px;
    cursor: pointer;

    button {
        padding: 0 !important;
    }
}

.edit-map {
    content: url('../assets/edit-icon.svg');
    margin: 0 auto;
    margin-bottom: 20px;
}

.add-map {
    content: url('../assets/add.svg');
    margin: 0 auto;
    margin-bottom: 20px;
}

input[type='number'] {
    width: 76px;
}

:deep(rv-basemap-item .rv-basemap-thumb img) {
    max-width: none;
}

:deep(.rv-details-attrib-value a) {
    white-space: unset !important;
}

$font-list:
    'Montserrat',
    -apple-system,
    BlinkMacSystemFont,
    Segoe UI,
    Helvetica,
    Arial,
    sans-serif,
    Apple Color Emoji,
    Segoe UI Emoji;

:deep(.ramp-app) {
    height: 100%;
    h1,
    h2,
    h3,
    h4,
    h5,
    h6,
    .h1,
    .h2,
    .h3,
    .h4,
    .h5,
    .h6 {
        font-family: $font-list;
        line-height: 1.5;
    }

    input[type='checkbox'] {
        margin-top: unset;
    }
}
</style>
