<template>
    <div
        class="hu-color-picker"
        :class="{ light: isLightTheme }"
        :style="{ width: totalWidth + 'px' }"
    >
        <div class="color-set">
            <Saturation
                ref="saturation"
                :color="rgbString"
                :hsv="hsv"
                :size="hueHeight"
                @selectSaturation="selectSaturation"
            />
            <Hue
                ref="hue"
                :hsv="hsv"
                :width="hueWidth"
                :height="hueHeight"
                @selectHue="selectHue"
            />
            <Alpha
                ref="alpha"
                :color="rgbString"
                :rgba="rgba"
                :width="hueWidth"
                :height="hueHeight"
                @selectAlpha="selectAlpha"
            />
        </div>
        <div
            :style="{ height: previewHeight + 'px' }"
            class="color-show"
        >
            <Preview
                :color="rgbaString"
                :width="previewWidth"
                :height="previewHeight"
            />
            <Sucker
                v-if="!suckerHide"
                :sucker-canvas="suckerCanvas"
                :sucker-area="suckerArea"
                @openSucker="openSucker"
                @selectSucker="selectSucker"
            />
        </div>
        <Box
            name="HEX"
            :color="modelHex"
            @inputColor="inputHex"
        />
        <Box
            name="RGBA"
            :color="modelRgba"
            @inputColor="inputRgba"
        />
        <Box
            name="RAL"
            :color="modelRal"
            @inputColor="inputRal"
        />
        <Colors
            :color="rgbaString"
            :colors-default="colorsDefault"
            :colors-history-key="colorsHistoryKey"
            @selectColor="selectColor"
        />
    </div>
</template>

<script>
import mixin from './mixin'
import Saturation from './Saturation.vue'
import Hue from './Hue.vue'
import Alpha from './Alpha.vue'
import Preview from './Preview.vue'
import Sucker from './Sucker.vue'
import Box from './Box.vue'
import Colors from './Colors.vue'
import { HEX2RAL } from '../schema/Classic.js';

export default {
    components: {
        Saturation,
        Hue,
        Alpha,
        Preview,
        Sucker,
        Box,
        Colors
    },
    mixins: [mixin],
    props: {
        color: {
            type: String,
            default: '#000000'
        },
        theme: {
            type: String,
            default: 'dark'
        },
        suckerHide: {
            type: Boolean,
            default: true
        },
        suckerCanvas: {
            type: null, // HTMLCanvasElement
            default: null
        },
        suckerArea: {
            type: Array,
            default: () => []
        },
        colorsDefault: {
            type: Array,
            default: () => [
                '#000000', '#FFFFFF', '#FF1900', '#F47365', '#FFB243', '#FFE623', '#6EFF2A', '#1BC7B1',
                '#00BEFF', '#2E81FF', '#5D61FF', '#FF89CF', '#FC3CAD', '#BF3DCE', '#8E00A7', 'rgba(0,0,0,0)'
            ]
        },
        colorsHistoryKey: {
            type: String,
            default: 'vue-colorpicker-history'
        }
    },
    data() {
        return {
            HEX2RAL,
            hueWidth: 15,
            hueHeight: 152,
            previewHeight: 30,
            modelRgba: '',
            modelHex: '',
            modelRal: '',
            r: 0,
            g: 0,
            b: 0,
            a: 1,
            h: 0,
            s: 0,
            v: 0
        }
    },
    computed: {
        isLightTheme() {
            return this.theme === 'light'
        },
        totalWidth() {
            return this.hueHeight + (this.hueWidth + 8) * 2
        },
        previewWidth() {
            return this.totalWidth - (this.suckerHide ? 0 : this.previewHeight)
        },
        rgba() {
            return {
                r: this.r,
                g: this.g,
                b: this.b,
                a: this.a
            }
        },
        hsv() {
            return {
                h: this.h,
                s: this.s,
                v: this.v
            }
        },
        rgbString() {
            return `rgb(${this.r}, ${this.g}, ${this.b})`
        },
        rgbaStringShort() {
            return `${this.r}, ${this.g}, ${this.b}, ${this.a}`
        },
        rgbaString() {
            return `rgba(${this.rgbaStringShort})`
        },
        hexString() {
            return this.rgb2hex(this.rgba, true)
        },
        ralString() {
            const ralValue = this.HEX2RAL[this.hexString];
            return ralValue ? ralValue.replace(/^(RAL)(\d{4})$/, '$1 $2') : '';
        }
    },
    created() {
        Object.assign(this, this.setColorValue(this.color))
        this.setText()

        // 避免初始化时，也会触发changeColor事件
        this.$watch('rgba', () => {
            this.$emit('changeColor', {
                rgba: this.rgba,
                hsv: this.hsv,
                hex: this.modelHex
            })
        })
    },
    methods: {
        selectSaturation(color) {
            const { r, g, b, h, s, v } = this.setColorValue(color)
            Object.assign(this, { r, g, b, h, s, v })
            this.setText()
        },
        selectHue(color) {
            const { r, g, b, h, s, v } = this.setColorValue(color)
            Object.assign(this, { r, g, b, h, s, v })
            this.setText()
            this.$nextTick(() => {
                this.$refs.saturation.renderColor()
                this.$refs.saturation.renderSlide()
            })
        },
        selectAlpha(a) {
            this.a = a
            this.setText()
        },
        inputHex(color) {
            const { r, g, b, a, h, s, v } = this.setColorValue(color)
            Object.assign(this, { r, g, b, a, h, s, v })
            this.modelHex = color;
            this.modelRgba = this.rgbaStringShort;
            this.modelRal = this.ralString;
            this.$nextTick(() => {
                this.$refs.saturation.renderColor()
                this.$refs.saturation.renderSlide()
                this.$refs.hue.renderSlide()
            })
        },
        inputRal(color) {
            const RALRegex = /^(?:RAL\s?)?(\d{4})$/i;
            const match = color.match(RALRegex);
            if (!match) {
                return;
            }
            // Normalize RAL colors to the format RAL XXXX
            const RALNormalized = `RAL${match[1]}`;

            const hex = this.getRalToHex(RALNormalized);
            if (!hex) {
                return;
            }

            const { r, g, b, a, h, s, v } = this.setColorValue(hex);
            Object.assign(this, { r, g, b, a, h, s, v });
            this.setText();
            this.$nextTick(() => {
                this.$refs.saturation.renderColor();
                this.$refs.saturation.renderSlide();
                this.$refs.hue.renderSlide();
            });
        },
        getRalToHex(normalizedRAL) {
            for (const hex in this.HEX2RAL) {
                if (this.HEX2RAL[hex] === normalizedRAL) {
                    return hex;
                }
            }
        },
        inputRgba(color) {
            const { r, g, b, a, h, s, v } = this.setColorValue(color)
            Object.assign(this, { r, g, b, a, h, s, v })
            this.modelHex = this.hexString;
            this.modelRgba = color;
            this.modelRal = this.ralString;
            this.$nextTick(() => {
                this.$refs.saturation.renderColor()
                this.$refs.saturation.renderSlide()
                this.$refs.hue.renderSlide()
            })
        },
        setText() {
            this.modelHex = this.hexString
            this.modelRgba = this.rgbaStringShort
            this.modelRal = this.ralString;
        },
        openSucker(isOpen) {
            this.$emit('openSucker', isOpen)
        },
        selectSucker(color) {
            const { r, g, b, a, h, s, v } = this.setColorValue(color)
            Object.assign(this, { r, g, b, a, h, s, v })
            this.setText()
            this.$nextTick(() => {
                this.$refs.saturation.renderColor()
                this.$refs.saturation.renderSlide()
                this.$refs.hue.renderSlide()
            })
        },
        selectColor(color) {
            const { r, g, b, a, h, s, v } = this.setColorValue(color)
            Object.assign(this, { r, g, b, a, h, s, v })
            this.setText()
            this.$nextTick(() => {
                this.$refs.saturation.renderColor()
                this.$refs.saturation.renderSlide()
                this.$refs.hue.renderSlide()
            })
        }
    }
}
</script>

<style lang="scss">
.hu-color-picker {
    padding: 10px;
    background: #1d2024;
    border-radius: 4px;
    box-shadow: 0 0 16px 0 rgba(0, 0, 0, 0.16);
    z-index: 1;
    &.light {
        background: #f7f8f9;
        .color-show {
            .sucker {
                background: #eceef0;
            }
        }
        .color-type {
            .name {
                background: #e7e8e9;
            }
            .value {
                color: #666;
                background: #eceef0;
            }
        }
        .colors.history {
            border-top: 1px solid #eee;
        }
    }
    canvas {
        vertical-align: top;
    }
    .color-set {
        display: flex;
    }
    .color-show {
        margin-top: 8px;
        display: flex;
    }
}
</style>
