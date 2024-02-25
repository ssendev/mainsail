<style scoped></style>

<template>
    <panel
        v-if="socketIsConnected"
        :icon="mdiWebcam"
        :title="$t('Panels.WebcamPanel.Headline')"
        :collapsible="$route.fullPath !== '/cam' && !bgCam && webcams.length"
        card-class="webcam-panel">
        <template v-if="webcams.length > 1" #buttons>
            <v-menu :offset-y="true" title="Cam Mode" v-if="currentPage == 'dashboard'">
                <template #activator="{ on, attrs }">
                    <v-btn text tile v-bind="attrs" v-on="on">
                        <v-icon small class="mr-2">{{ mdiImageSizeSelectActual }}</v-icon>
                        <v-icon small>{{ mdiMenuDown }}</v-icon>
                    </v-btn>
                </template>
                <v-list dense class="py-0">
                    <v-list-item link @click="bgCam = !bgCam">
                        <template>
                            <v-list-item-icon class="mr-2">
                                <v-icon small class="mt-1">{{ bgCam ? mdiCheckboxMarked : mdiCheckboxBlankOutline }}</v-icon>
                            </v-list-item-icon>
                            <v-list-item-content>
                                <v-list-item-title>{{ $t('Panels.WebcamPanel.Background') }}</v-list-item-title>
                            </v-list-item-content>
                        </template>
                    </v-list-item>
                    <template v-if="bgCam">
                        <v-list-item link @click="bgContain = !bgContain">
                            <template>
                                <v-list-item-icon class="mr-2">
                                    <v-icon small class="mt-1">{{ bgContain ? mdiCheckboxMarked : mdiCheckboxBlankOutline }}</v-icon>
                                </v-list-item-icon>
                                <v-list-item-content>
                                    <v-list-item-title>{{ $t('Panels.WebcamPanel.Contain') }}</v-list-item-title>
                                </v-list-item-content>
                            </template>
                        </v-list-item>
                        <v-list-item link class="px-0" :ripple="false">
                            <template>
                                <v-list-item-content>
                                    <v-list-item-title class="pl-12 pr-4">{{ $t('Panels.WebcamPanel.Alpha') }}</v-list-item-title>
                                    <v-list-item-subtitle class="px-4">
                                        <v-slider
                                            v-model="bgAlpha"
                                            :min="0"
                                            :max="100"
                                            :step="10"
                                            hide-details
                                            >
                                        </v-slider>
                                    </v-list-item-subtitle>
                                </v-list-item-content>
                            </template>
                        </v-list-item>
                    </template>
                </v-list>
                <v-sheet v-if="bgCam">
                    <v-divider></v-divider>
                    <v-item-group v-model="bgAlign" mandatory>
                        <div class="d-flex flex-wrap">
                            <v-item v-for="icon, i in bgAlignments" :key="i">
                                <v-btn icon tile :outlined="i == bgAlign" @click="bgAlign = i" style="width: 33%;">
                                    <v-icon small class="mt-1">{{ icon }}</v-icon>
                                </v-btn>
                            </v-item>
                        </div>
                    </v-item-group>
                </v-sheet>
            </v-menu>
            <v-menu :offset-y="true" title="Webcam">
                <template #activator="{ on, attrs }">
                    <v-btn text tile v-bind="attrs" v-on="on">
                        <v-icon v-if="'icon' in currentCam" small class="mr-2">
                            {{ convertWebcamIcon(currentCam.icon) }}
                        </v-icon>
                        <span class="d-none d-md-block">{{ 'name' in currentCam ? currentCam.name : 'unknown' }}</span>
                        <v-icon small>{{ mdiMenuDown }}</v-icon>
                    </v-btn>
                </template>
                <v-list dense class="py-0">
                    <v-list-item link @click="currentCamId = 'all'">
                        <v-list-item-icon class="mr-2">
                            <v-icon small class="mt-1">{{ mdiViewGrid }}</v-icon>
                        </v-list-item-icon>
                        <v-list-item-content>
                            <v-list-item-title>{{ $t('Panels.WebcamPanel.All') }}</v-list-item-title>
                        </v-list-item-content>
                    </v-list-item>
                    <v-list-item v-for="webcam of webcams" :key="webcam.id" link @click="currentCamId = webcam.id">
                        <v-list-item-icon class="mr-2">
                            <v-icon small class="mt-1">{{ convertWebcamIcon(webcam.icon) }}</v-icon>
                        </v-list-item-icon>
                        <v-list-item-content>
                            <v-list-item-title v-text="webcam.name"></v-list-item-title>
                        </v-list-item-content>
                    </v-list-item>
                </v-list>
            </v-menu>
        </template>
        <v-card-text v-if="webcams.length && (currentPage != 'dashboard' || !bgCam)" class="px-0 py-0 content d-inline-block">
            <v-row>
                <v-col class="pb-0" style="position: relative">
                    <template v-if="currentCam.service === 'grid'">
                        <webcam-grid :webcams="webcams"></webcam-grid>
                    </template>
                    <template v-else-if="currentCam.service === 'mjpegstreamer'">
                        <webcam-mjpegstreamer :cam-settings="currentCam"></webcam-mjpegstreamer>
                    </template>
                    <template v-else-if="currentCam.service === 'mjpegstreamer-adaptive'">
                        <webcam-mjpegstreamer-adaptive :cam-settings="currentCam"></webcam-mjpegstreamer-adaptive>
                    </template>
                    <template v-else-if="currentCam.service === 'uv4l-mjpeg'">
                        <webcam-uv4l-mjpeg :cam-settings="currentCam"></webcam-uv4l-mjpeg>
                    </template>
                    <template v-else-if="currentCam.service === 'ipstream'">
                        <webcam-ipstreamer :cam-settings="currentCam"></webcam-ipstreamer>
                    </template>
                    <template v-else>
                        <p class="text-center py-3 font-italic">{{ $t('Panels.WebcamPanel.UnknownWebcamService') }}</p>
                    </template>
                </v-col>
            </v-row>
        </v-card-text>
        <portal v-else-if="webcams.length && bgCam">
            <div :style="`position: fixed; top: ${topbarHeight}px; left: ${naviDrawer ? navigationWidth : 0}px; right: 0; bottom: 0; z-index: -1;`" class="v-application theme--dark">
                <div style="height: 100%; width: 100%; overflow: hidden;" :class="`d-flex align-${bgVertical} justify-${bgHirizontal}`">
                    <div :style="bgSize" class="flex-shrink-0 flex-grow-0">
                        <template v-if="currentCam.service === 'grid'">
                            <webcam-grid :webcams="webcams" class="pa-0" style="padding-bottom: 0 !important;"></webcam-grid>
                        </template>
                        <template v-else-if="currentCam.service === 'mjpegstreamer'">
                            <webcam-mjpegstreamer :cam-settings="currentCam"></webcam-mjpegstreamer>
                        </template>
                        <template v-else-if="currentCam.service === 'mjpegstreamer-adaptive'">
                            <webcam-mjpegstreamer-adaptive :cam-settings="currentCam"></webcam-mjpegstreamer-adaptive>
                        </template>
                        <template v-else-if="currentCam.service === 'uv4l-mjpeg'">
                            <webcam-uv4l-mjpeg :cam-settings="currentCam"></webcam-uv4l-mjpeg>
                        </template>
                        <template v-else-if="currentCam.service === 'ipstream'">
                            <webcam-ipstreamer :cam-settings="currentCam"></webcam-ipstreamer>
                        </template>
                        <template v-else>
                            <p class="text-center py-3 font-italic">{{ $t('Panels.WebcamPanel.UnknownWebcamService') }}</p>
                        </template>
                    </div>
                </div>
            </div>
        </portal>
        <v-card-text v-else>
            <p class="text-center mb-0 text--disabled">{{ $t('Panels.WebcamPanel.NoWebcam') }}</p>
        </v-card-text>
    </panel>
</template>

<script lang="ts">
import Mjpegstreamer from '@/components/webcams/Mjpegstreamer.vue'
import MjpegstreamerAdaptive from '@/components/webcams/MjpegstreamerAdaptive.vue'
import Ipstreamer from '@/components/webcams/Ipstreamer.vue'
import Uv4lMjpeg from '@/components/webcams/Uv4lMjpeg.vue'
import WebcamGrid from '@/components/webcams/WebcamGrid.vue'
import Component from 'vue-class-component'
import { Mixins, Prop } from 'vue-property-decorator'
import BaseMixin from '../mixins/base'
import Panel from '@/components/ui/Panel.vue'
import { GuiWebcamStateWebcam } from '@/store/gui/webcams/types'
import { mdiMenuDown, mdiViewGrid, mdiWebcam, mdiImageSizeSelectActual, mdiCircleOpacity, mdiCheckboxBlankOutline, mdiCheckboxMarked, mdiArrowTopLeft, mdiArrowUp, mdiArrowTopRight, mdiArrowLeft, mdiArrowRight, mdiArrowBottomLeft, mdiArrowDown, mdiArrowBottomRight } from '@mdi/js'
import WebcamMixin from '@/components/mixins/webcam'
import { CamBackground } from '@/store/gui/types'
import { Portal } from '@linusborg/vue-simple-portal'
import { navigationWidth, topbarHeight } from '@/store/variables'

@Component({
    components: {
        Panel,
        Portal,
        'webcam-mjpegstreamer': Mjpegstreamer,
        'webcam-mjpegstreamer-adaptive': MjpegstreamerAdaptive,
        'webcam-ipstreamer': Ipstreamer,
        'webcam-uv4l-mjpeg': Uv4lMjpeg,
        'webcam-grid': WebcamGrid,
    },
})
export default class WebcamPanel extends Mixins(BaseMixin, WebcamMixin) {
    @Prop({ default: 'dashboard' }) declare currentPage?: string

    navigationWidth = navigationWidth
    topbarHeight = topbarHeight
    mdiWebcam = mdiWebcam
    mdiMenuDown = mdiMenuDown
    mdiViewGrid = mdiViewGrid
    mdiImageSizeSelectActual = mdiImageSizeSelectActual
    // mdiCircleOpacity = mdiCircleOpacity
    mdiCheckboxBlankOutline = mdiCheckboxBlankOutline
    mdiCheckboxMarked = mdiCheckboxMarked

    private bgAlignments = [ mdiArrowTopLeft, mdiArrowUp, mdiArrowTopRight, mdiArrowLeft, mdiImageSizeSelectActual, mdiArrowRight, mdiArrowBottomLeft, mdiArrowDown, mdiArrowBottomRight ]

    private bgModes = ['Panel', 'Background']
    // private bgContain = true
    private bgAspect = 1.33333

    get naviDrawer(): boolean {
        return this.$store.state.naviDrawer
    }

    get bgCam() {
        return this.camBackground.enabled
    }

    set bgCam(enabled) {
        this.$store.dispatch('gui/saveSetting', { name: `dashboardCamBackgrounds.${this.viewport}.enabled`, value: enabled })
    }

    get bgContain() {
        return this.camBackground.contain
    }

    set bgContain(contain) {
        this.$store.dispatch('gui/saveSetting', { name: `dashboardCamBackgrounds.${this.viewport}.contain`, value: contain })
    }

    get bgAlpha() {
        return this.camBackground.alpha
    }

    set bgAlpha(alpha) {
        this.$store.dispatch('gui/saveSetting', { name: `dashboardCamBackgrounds.${this.viewport}.alpha`, value: alpha })
    }

    get bgAlign() {
        return this.camBackground.align
    }

    set bgAlign(align) {
        this.$store.dispatch('gui/saveSetting', { name: `dashboardCamBackgrounds.${this.viewport}.align`, value: align })
    }

    get camBackground(): CamBackground {
        return this.$store.state.gui.dashboardCamBackgrounds[this.viewport]
    }

    get bgHirizontal() {
        const align = this.bgAlign % 3
        if (align == 0) {
            return 'begin'
        } else if (align == 1) {
            return 'center'
        } else {
            return 'end'
        }
    }

    get bgVertical() {
        const align = Math.floor(this.bgAlign / 3)
        if (align == 0) {
            return 'begin'
        } else if (align == 1) {
            return 'center'
        } else {
            return 'end'
        }
    }

    get bgSize() {
        if (this.bgContain) {
            return `width: 100%; max-width: calc((100vh - 48px) * ${this.bgAspect})`
        } else {
            return `width: max(100%, calc((100vh - 48px) * ${this.bgAspect}))`
        }
    }

    get webcams(): GuiWebcamStateWebcam[] {
        return this.$store.getters['gui/webcams/getWebcams']
    }

    get currentCamId(): string {
        if (this.webcams.length === 1) return this.webcams[0].id ?? 'all'

        let currentCamId = this.$store.state.gui.view.webcam.currentCam[this.currentPage ?? ''] ?? 'all'
        if (this.webcams.findIndex((webcam: GuiWebcamStateWebcam) => webcam.id === currentCamId) !== -1)
            return currentCamId
        else if (currentCamId !== undefined && this.webcams.length === 1) return this.webcams[0].id ?? ''
        else return 'all'
    }

    set currentCamId(newVal: string) {
        this.$store.dispatch('gui/setCurrentWebcam', { page: this.currentPage, value: newVal })
    }

    get currentCam(): any {
        const cam = this.webcams.find((cam: GuiWebcamStateWebcam) => cam.id === this.currentCamId)

        return (
            cam ?? {
                name: this.$t('Panels.WebcamPanel.All'),
                service: 'grid',
                icon: mdiViewGrid,
            }
        )
    }
}
</script>
