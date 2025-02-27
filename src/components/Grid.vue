<script>
// Sets up all layers/overlays for the grid with 'grid_id'

import Grid from './js/grid.js'
import Canvas from '../mixins/canvas.js'
import Crosshair from './Crosshair.vue'
import KeyboardListener from './KeyboardListener.vue'

import Spline from "./overlays/Spline.vue"
import Splines from "./overlays/Splines.vue"
import Range from "./overlays/Range.vue"
import Trades from "./overlays/Trades.vue"
import Channel from "./overlays/Channel.vue"
import Segment from "./overlays/Segment.vue"
import Candles from "./overlays/Candles.vue"
import Volume from "./overlays/Volume.vue"
import Splitters from "./overlays/Splitters.vue"
import LineTool from "./overlays/LineTool.vue"
import RangeTool from "./overlays/RangeTool.vue"


export default {
    name: 'Grid',
    props: [
        'sub', 'layout', 'range', 'interval', 'cursor', 'colors', 'overlays',
        'width', 'height', 'data', 'grid_id', 'y_transform', 'font', 'tv_id',
        'config', 'meta'
    ],
    mixins: [Canvas],
    components: { Crosshair, KeyboardListener },
    created() {
        // List of all possible overlays (builtin + custom)
        this._list = [
            Spline, Splines, Range, Trades, Channel, Segment,
            Candles, Volume, Splitters, LineTool, RangeTool
        ]
        .concat(this.$props.overlays)
        this._registry = {}

        // We need to know which components we will use.
        // Custom overlay components overwrite built-ins:
        let tools = []
        this._list.forEach((x, i) => {
            let use_for = x.methods.use_for()
            if (x.methods.tool) tools.push({
                use_for, info: x.methods.tool()
            })
            use_for.forEach(indicator => {
                this._registry[indicator] = i
            })
        })
        this.$emit('custom-event', {
            event: 'register-tools', args: tools
        })
    },
    mounted() {
        const el = this.$refs['canvas']
        this.renderer = new Grid(el, this)
        this.setup()
        this.$nextTick(() => this.redraw())
    },
    render(h) {
        const id = this.$props.grid_id
        const layout = this.$props.layout.grids[id]
        return this.create_canvas(h, `grid-${id}`, {
            position: {
                x: 0,
                y: layout.offset || 0
            },
            attrs: {
                width: layout.width,
                height: layout.height
            },
            style: {
                backgroundColor: this.$props.colors.colorBack
            },
            hs: [
                h(Crosshair, {
                    props: this.common_props(),
                    on: this.layer_events
                }),
                h(KeyboardListener, {
                    on: this.keyboard_events
                }),
            ].concat(this.get_overlays(h))
        })
    },
    methods: {
        new_layer(layer) {
            this.$nextTick(() => this.renderer.new_layer(layer))
        },
        del_layer(layer) {
            this.$nextTick(() => this.renderer.del_layer(layer))
            const grid_id = this.$props.grid_id
            this.$emit('custom-event', {
                event: 'remove-shaders',
                args: [grid_id, layer]
            })
            this.$emit('custom-event', {
                event: 'remove-layer-meta',
                args: [grid_id, layer]
            })
        },
        get_overlays(h) {
            // Distributes overlay data & settings according
            // to this._registry; returns compo list
            let comp_list = [], count = {}

            for (var d of this.$props.data) {
                let comp = this._list[this._registry[d.type]]
                if (comp) {
                    comp_list.push({
                        cls: comp,
                        type: d.type,
                        data: d.data,
                        settings: d.settings
                    })
                    count[d.type] = 0
                }
            }
            return comp_list.map((x, i) => h(x.cls, {
                    on: this.layer_events,
                    attrs: Object.assign(this.common_props(), {
                        id: `${x.type}_${count[x.type]++}`,
                        type: x.type,
                        data: x.data,
                        settings: x.settings,
                        num: i,
                        grid_id: this.$props.grid_id,
                        meta: this.$props.meta
                    })
                })
            )
        },
        common_props() {
            return {
                cursor: this.$props.cursor,
                colors: this.$props.colors,
                layout: this.$props.layout.grids[this.$props.grid_id],
                interval: this.$props.interval,
                sub: this.$props.sub,
                font: this.$props.font,
                config: this.$props.config,
            }
        }
    },
    computed: {
        is_active() {
            return this.$props.cursor.t !== undefined &&
            this.$props.cursor.grid_id === this.$props.grid_id
        }
    },
    watch: {
        range: {
            handler: function() {
                // TODO: Left-side render lag fix:
                // Overlay data is updated one tick later than
                // the main sub. Fast fix is to delay redraw()
                // call. It will be a solution until a better
                // one comes by.
                this.$nextTick(() => this.redraw())
            },
            deep: true
        },
        cursor: {
            handler: function() {
                if (!this.$props.cursor.locked) this.redraw()
            },
            deep: true
        }
    },
    data() {
        return {
            layer_events: {
                'new-grid-layer': this.new_layer,
                'delete-grid-layer': this.del_layer,
                'show-grid-layer': d => {
                    this.renderer.show_hide_layer(d)
                    this.redraw()
                },
                'redraw-grid': this.redraw,
                'layer-meta-props': d => this.$emit('layer-meta-props', d),
                'custom-event': d => this.$emit('custom-event', d)
            },
            keyboard_events: {
                'register-kb-listener': event => {
                    this.$emit('register-kb-listener', event)
                },
                'remove-kb-listener': event => {
                    this.$emit('remove-kb-listener', event)
                },
                'keyup': event => {
                    if (!this.is_active) return
                    this.renderer.propagate('keyup', event)
                },
                'keydown': event => {
                    if (!this.is_active) return
                    this.renderer.propagate('keydown', event)
                },
                'keypress': event => {
                    if (!this.is_active) return
                    this.renderer.propagate('keypress', event)
                },
            }
        }
    }
}

</script>
