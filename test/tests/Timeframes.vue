<template>
<div>
    <trading-vue :data="chart" :width="this.width" :height="this.height"
            :color-back="colors.colorBack"
            :color-grid="colors.colorGrid"
            :color-text="colors.colorText"
            ref="tradingVue">
    </trading-vue>
    <tf-selector :charts="charts" v-on:selected="on_selected">
    </tf-selector>
</div>
</template>

<script>
import TradingVue from '../../src/TradingVue.vue'
import TfSelector from './Timeframes/TFSelector.vue'
import Data from '../data/data_tf.json'
import Utils from '../../src/stuff/utils.js'
import DataCube from '../../src/helpers/datacube.js'


export default {
    name: 'Timeframes',
    description: 'Should display correct dates for every timeframe',
    components: {
        TradingVue, TfSelector
    },
    methods: {
        onResize(event) {
            this.width = window.innerWidth
            this.height = window.innerHeight - 50
        },
        on_selected(tf) {
            // TODO: This does not update the interval:
            // this.chart.set('chart.data', this.charts[tf.name])

            // Works only with a full update:
            this.chart = new DataCube({
                ohlcv: this.charts[tf.name]
            })
            this.$refs.tradingVue.resetChart()
        }
    },
    mounted() {
        window.addEventListener('resize', this.onResize)
        this.onResize()
        window.DataCube = this.chart // Debug
    },
    beforeDestroy() {
        window.removeEventListener('resize', this.onResize)
    },
    data() {
        return {
            charts: Data,
            chart: new DataCube({}),
            width: window.innerWidth,
            height: window.innerHeight,
            colors: {
                colorBack: '#fff',
                colorGrid: '#eee',
                colorText: '#333',
            }
        };
    }
};
</script>

<style>

</style>
