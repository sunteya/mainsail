<style lang="scss">
    //  >  > table
.v-data-table.table-adjust {
    > .v-data-table__wrapper > table {
        border-collapse: collapse;
        width: auto;
        margin-left: auto;
        margin-right: auto;
    }

    tr:hover {
        background-color: transparent !important;
    }

    th {
        width: 1em !important;
        border-width: 0 !important;
        height: auto !important;
    }

    td {
        border: thin solid rgba(255, 255, 255, 0.12);
        cursor: pointer;
        position: relative;
        width: 5em;


        > .inner {
            position: relative;
            z-index: 0;
        }

        > .bg {
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            z-index: 0;
        }

        &.active {
            > .bg {
                background-color: var(--v-primary-base);
                opacity: 0.05;
            }
        }
    }
}
</style>

<template>
    <panel
        v-if="currentProfile"
        :icon="mdi.mdiVectorSquareEdit"
        :title="`Mesh Adjust [${currentProfile.name}]`"
        :collapsible="true"
        card-class="">

        <template #buttons>
            <v-btn text tile @click="test">Test</v-btn>
            <textarea ref="clipboard-input" style="position: absolute; z-index: -1; width: 1px; height: 1px; opacity: 0;"></textarea>
            <v-btn text tile @click="exportPoints">Export</v-btn>
        </template>

        <v-container>
            <div class="d-flex flex-nowrap">
                <div class="text--secondary">
                    <span class="me-3">
                        <span class="text-no-wrap mr-2">XY ({{ gcodePositions.x }}, {{ gcodePositions.y }})</span>
                        <span class="text-no-wrap">Z ({{ gcodePositions.z }} + {{ zOffset }})</span>
                    </span>
                </div>

                <div class="ms-auto d-flex">
                    <span v-if="isSingularity" class="me-3">
                        <v-icon small class="me-1">{{ mdi.mdiNumeric0CircleOutline }}</v-icon>
                        <span class="text--secondary">{{zOffset}}</span>
                    </span>
                    <span v-else class="me-3">
                        <v-icon small class="me-1">{{ mdi.mdiSwapVertical }}</v-icon>
                        <span class="text--secondary">{{activeOffset}}</span>
                    </span>

                    <div class="me-2 v-item-group theme--dark v-btn-toggle">
                        <v-btn v-for="(offset, index) in [ -0.02, -0.01, -0.005 ]" :key="index" small @click="changeOffset(offset)">
                            <span>{{offset}}</span>
                            <v-icon v-if="index == 2" small right>{{mdi.mdiArrowCollapseDown}}</v-icon>
                        </v-btn>
                    </div>

                    <div class="me-2 v-item-group theme--dark v-btn-toggle">
                        <v-btn v-for="(offset, index) in [ 0.005, 0.01, 0.02 ]" :key="index" active-class="foo" small @click="changeOffset(offset)">
                            <v-icon v-if="index == 0" small left>{{mdi.mdiArrowCollapseUp}}</v-icon>
                            <span>{{ offset }}</span>
                        </v-btn>
                    </div>
                </div>
            </div>
        </v-container>

        <v-container>
            <v-simple-table class="table-adjust">
                <tbody>
                    <tr>
                        <th class="pa-0"></th>
                        <th v-for="(x_pos, x_index) in currentProfile.data.x_count" :key="x_index" class="pa-0 text-center secondary--text">{{ (currentMeshInfo.min_x + currentMeshInfo.distance_x * x_index).toFixed(1) }}</th>
                        <th class="pa-0"></th>
                    </tr>
                    <tr v-for="y_index in currentMeshInfo?.axis_y" :key="y_index">
                        <th class="pa-0 pe-2 text-right secondary--text">{{ y_index }}</th>
                        <td
                            v-for="(x_pos, x_index) in currentProfile.data.x_count"
                            :key="x_pos"
                            class="text-center pa-0"
                            :class="{ 'active': x_index == activeIndexX && y_index == activeIndexY }"
                            @click="moveToIndex(x_index, y_index, true)">

                            <div class="inner">
                                <template v-if="x_index == currentMeshInfo?.singularity_x_index && y_index == currentMeshInfo?.singularity_y_index">
                                    <v-icon small>{{ mdi.mdiNumeric0CircleOutline }}</v-icon>
                                </template>
                                <span v-else-if="offsetMatrix[y_index] == null || offsetMatrix[y_index][x_index] == null">
                                </span>
                                <span v-else :class="{ 'success--text': (offsetMatrix[y_index][x_index] || 0) > 0, 'error--text': (offsetMatrix[y_index][x_index] || 0) < 0 }">
                                    {{ offsetMatrix[y_index][x_index]?.toFixed(3) }}
                                </span>
                            </div>
                            <div class="bg"></div>
                        </td>
                        <th class="pa-0 ps-2 secondary--text">{{ (currentMeshInfo.min_y + currentMeshInfo.distance_y * y_index).toFixed(1) }}</th>
                    </tr>
                    <tr>
                        <th class="pa-0"></th>
                        <th v-for="(x_pos, x_index) in currentProfile.data.x_count" :key="x_index" class="pa-0 text-center secondary--text">{{ x_index }}</th>
                        <th class="pa-0"></th>
                    </tr>
                </tbody>
            </v-simple-table>
        </v-container>
    </panel>
</template>

<script lang="ts">
import { Component, Mixins, Watch } from 'vue-property-decorator'
import BaseMixin from '../mixins/base'
import ControlMixin from '@/components/mixins/control'
import Panel from '@/components/ui/Panel.vue'
import * as mdi from '@mdi/js'
import { PrinterStateBedMesh } from '@/store/printer/types'
import axios from 'axios'

@Component({
    components: {
        Panel,
    },
})

export default class ToolheadControlPanel extends Mixins(BaseMixin, ControlMixin) {
    mdi = mdi

    activeIndexX: number | null = null
    activeIndexY: number | null = null
    offsetMatrix: { [y:number]: { [x:number]: number | null } } = {}

    get gcodePositions() {
        const pos = this.$store.state.printer.gcode_move?.gcode_position ?? [0, 0, 0]
        return {
            x: pos[0]?.toFixed(2) ?? '--',
            y: pos[1]?.toFixed(2) ?? '--',
            z: pos[2]?.toFixed(3) ?? '--',
        }
    }

    get zOffset(): number {
        return this.$store.state.printer?.gcode_move?.homing_origin[2].toFixed(3)
    }

    get profiles() {
        return this.$store.getters['printer/getBedMeshProfiles']
    }

    get bed_mesh() {
        return this.$store.state.printer.bed_mesh ?? null
    }

    get isSingularity() {
        if (!this.currentProfile) {
            return false
        }

        return this.activeIndexX == this.currentMeshInfo.singularity_x_index && this.activeIndexY == this.currentMeshInfo.singularity_y_index
    }

    @Watch('bed_mesh', { deep: true })
    bed_meshChanged() {
        this.reset()
    }

    mounted() {
        this.reset()
    }

    reset() {
        this.activeIndexX = null
        this.activeIndexY = null

        const matrix: { [y:number]: { [x:number]: number | null } } = {}

        if (this.currentProfile) {
            const data = this.currentProfile.data
            const count_x = data["x_count"]
            const count_y = data["y_count"]

            for (let index_y = 0; index_y < count_y; index_y++) {
                matrix[index_y] = {}
                for (let index_x = 0; index_x < count_x; index_x++) {
                    matrix[index_y][index_x] = null
                }
            }
        }

        this.offsetMatrix = matrix
    }

    get activeOffset() {
        if (this.activeIndexX == null || this.activeIndexY == null) {
            return 0.0.toFixed(3)
        }

        const offset = this.offsetMatrix[this.activeIndexY][this.activeIndexX] || 0.0
        return offset.toFixed(3)
    }

    changeGcodeOffset(offset: number) {
        const gcode = `SET_GCODE_OFFSET Z_ADJUST=${offset} MOVE=1`
        this.$store.dispatch('server/addEvent', { message: gcode, type: 'command' })
        this.$socket.emit('printer.gcode.script', { script: gcode })
    }

    changeMeshOffset(index_x: number, index_y: number, num: number) {
        const offset = this.offsetMatrix[index_y][index_x] || 0
        this.offsetMatrix[index_y][index_x] = offset + num
        this.moveToIndex(index_x, index_y, false)
    }

    changeOffset(num: number) {
        if (this.isSingularity) {
            this.changeGcodeOffset(num)
        } else {
            this.changeMeshOffset(this.activeIndexX!, this.activeIndexY!, num)
        }
    }

    get currentMeshInfo() {
        const data = this.currentProfile.data
        const count_x = data["x_count"] as number
        const count_y = data["y_count"] as number

        const axis_y = []
        for (let index_y = 0; index_y < count_y; index_y++) {
            axis_y.push(index_y)
        }
        axis_y.reverse()

        return {
            min_x: data["min_x"] as number,
            min_y: data["min_y"] as number,
            count_x,
            count_y,
            axis_y: axis_y,
            distance_x: (data["max_x"] - data["min_x"]) / (count_x - 1),
            distance_y: (data["max_y"] - data["min_y"]) / (count_y - 1),
            singularity_x_index: Math.floor(count_x / 2),
            singularity_y_index: Math.floor(count_y / 2),
        }
    }

    get currentProfile() {
        const profileName = this.bed_mesh?.profile_name ?? ''
        return this.profiles.find((profile: PrinterStateBedMesh) => profile.name === profileName)
    }

    moveToIndex(index_x: number, index_y: number, lift: boolean) {
        this.activeIndexX = index_x
        this.activeIndexY = index_y

        const data = this.currentProfile.data
        const pointX = (data["min_x"] + this.currentMeshInfo!.distance_x * index_x).toFixed(1)
        const pointY = (data["min_y"] + this.currentMeshInfo!.distance_y * index_y).toFixed(1)
        const pointZ = 0

        const offsets = this.offsetMatrix[index_y] ||= []
        const offsetZ = offsets[index_x] || 0
        const speed = 150 * 60

        const gcodes = [
            `G1 X${pointX} Y${pointY} Z${(pointZ + offsetZ).toFixed(6)} F${speed}`,
        ]

        if (lift) {
            gcodes.unshift(`G1 X${pointX} Y${pointY} Z3 F${speed}`)
        }

        if (!this.absolute_coordinates) {
            gcodes.unshift("G90")
            gcodes.push("G91")
        }

        this.doSend(gcodes.join("\n"))
    }

    exportPoints() {
        const lines = [ "#*# points =" ]
        const points: number[][] = this.currentProfile.data.points

        for (let index_y = 0; index_y < this.currentMeshInfo.count_y; index_y++) {
            const row: string[] = []
            for (let index_x = 0; index_x < this.currentMeshInfo.count_x; index_x++) {
                const position = points[index_y][index_x]
                const offset = this.offsetMatrix[index_y][index_x] || 0
                const value = (position + offset).toFixed(6)
                row.push(value)
            }

            lines.push("#*# \t  " + row.join(", "))
        }

        const input = this.$refs["clipboard-input"] as any
        input.value = lines.join("\n")
        input.select();
        input.setSelectionRange(0, 99999)
        document.execCommand('copy')
    }

    async test() {
        const file = this.$store.getters['files/getFile']('config/printer.cfg')
        const href = `${this.apiUrl}/server/files/config/printer.cfg`
        // window.open(href)
        console.log(file)
        const resp = await axios.get(href)
        console.log(resp.data)
        console.log(resp)
    }
}
</script>

