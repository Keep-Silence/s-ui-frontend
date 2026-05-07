<template>
  <v-dialog transition="dialog-bottom-transition" width="800">
    <v-card class="rounded-lg" :loading="loading">
      <v-card-title>
        <v-row>
          <v-col cols="auto">
            {{ $t('stats.graphTitle') }}
          </v-col>
          <v-spacer></v-spacer>
          <v-col cols="auto"><v-icon icon="mdi-close" @click="$emit('close')"></v-icon></v-col>
        </v-row>
      </v-card-title>
      <v-divider></v-divider>
      <v-card-text style="padding: 0 16px;">
        <div style="text-align: center; margin: 5px;">
          {{ $t('objects.' + resource) + " : " + tag }}
        </div>
        <v-radio-group v-model="limit" @change="loadData" density="compact" :loading="loading" inline hide-details>
          <v-radio v-for="p in periods" :label="p.title" :value="p.value"></v-radio>
        </v-radio-group>
          <v-container id="container" style="height:40vh;">
            <v-skeleton-loader
            class="mx-auto border"
            width="95%"
            type="image"
            v-if="loading"
          ></v-skeleton-loader>
          <template v-else>
            <v-alert :text="$t('noData')" type="warning" variant="outlined" v-if="alert"></v-alert>
            <Line v-if="loaded" :data="usage" :options="<any>options" />
          </template>
        </v-container>
      </v-card-text>
    </v-card>
  </v-dialog>
</template>

<script lang="ts">
import { i18n } from '@/locales'
import HttpUtils from '@/plugins/httputil'
import { HumanReadable } from '@/plugins/utils'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler,
} from 'chart.js'
import { ref } from 'vue'
import { Line } from 'vue-chartjs'
ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
)
ChartJS.defaults.font.family = 'Vazirmatn'
export default {
  components: {
    Line
  },
  props: ['visible','resource','tag'],
  data() {
    return {
      loading: false,
      loaded: false,
      alert: false,
      intervalId: <any>0,
      limit: 1,
      periods: [
        { value: 1, title: i18n.global.n(1) + i18n.global.t('date.h')},
        { value: 6, title: i18n.global.n(6) + i18n.global.t('date.h')},
        { value: 12, title: i18n.global.n(12) + i18n.global.t('date.h')},
        { value: 24, title: i18n.global.n(1) + i18n.global.t('date.d')},
        { value: 48, title: i18n.global.n(2) + i18n.global.t('date.d')},
        { value: 240, title: i18n.global.n(10) + i18n.global.t('date.d')},
        { value: 480, title: i18n.global.n(20) + i18n.global.t('date.d')},
        { value: 720, title: i18n.global.n(30) + i18n.global.t('date.d')},
        { value: 1440, title: i18n.global.n(60) + i18n.global.t('date.d')},
        { value: 2160, title: i18n.global.n(90) + i18n.global.t('date.d')},
      ],
      options: {
        responsive: true,
        maintainAspectRatio: false,
        interaction: {
          intersect: false,
          mode: 'index',
        },
        elements: {
          point: { pointStyle: 'crossRot' }
        },
        plugins: {
          tooltip: {
            callbacks: {
              text: (ctx:any) => {
                const {axis = 'xy', intersect, mode} = ctx.chart.options.interaction
                return 'Mode: ' + mode + ', axis: ' + axis + ', intersect: ' + intersect
              },
              footer: (items:any[]) => {
                return HumanReadable.sizeFormat(items.reduce((acc, c) => acc + c.raw, 0))
              }
            }
          }
        },
        scales: {
          y: {
            grid: {
              color: '#777777',
            },
            beginAtZero: true,
            ticks: {
              callback: function(label:any, index: number) {
                return label == 0 ? 0 : HumanReadable.sizeFormat(label,0)
              },
              count: 10
            }
          }
        }
      },
      usage: ref(<any>{}),
    }
  },
  methods: {
    // 为不同 node 生成颜色
    getNodeColor(node: string, index: number, isUpload: boolean, isBackground: boolean = false) {
      // 预定义的颜色数组
      const colors = [
        { up: [255, 165, 0], down: [0, 128, 0] },     // 橙/绿
        { up: [255, 0, 0], down: [0, 0, 255] },       // 红/蓝
        { up: [128, 0, 128], down: [0, 128, 128] },   // 紫/青
        { up: [255, 192, 203], down: [75, 0, 130] },  // 粉/靛
        { up: [255, 140, 0], down: [32, 178, 170] },  // 深橙/海绿
        { up: [220, 20, 60], down: [30, 144, 255] },  // 猩红/道奇蓝
      ]
      const colorSet = colors[index % colors.length]
      const rgb = isUpload ? colorSet.up : colorSet.down
      const alpha = isBackground ? (isUpload ? 0.4 : 0.2) : 1
      return `rgba(${rgb[0]}, ${rgb[1]}, ${rgb[2]}, ${alpha})`
    },

    async loadData() {
      this.loading = true
      const data = await HttpUtils.get('api/stats', { resource: this.resource, tag: this.tag, limit: this.limit })
      if (data.success && data.obj) {
        const obj = <any[]>data.obj
        const l = String(i18n.global.locale) == 'fa' ? "fa-IR" : "en-US"
        const oneStep = this.limit * 3600 * 1000 / 360 // Each 10 sec
        const now = new Date().getTime()
        const steps = <number[]>[]
        for (let i = 360; i >= 0; i--) {
          steps.push(now - (oneStep * i))
        }
        const labels = <string[]>[]

        // 获取所有唯一的 node 值
        const nodes = [...new Set(obj.map(o => o.node).filter(n => n))]

        // 为每个 node 创建上传和下载数据数组
        const nodeData = new Map<string, { uplink: (number | null)[], downlink: (number | null)[] }>()
        nodes.forEach(node => {
          nodeData.set(node, { uplink: [], downlink: [] })
        })

        for (let i = 1; i < 360; i++) {
          labels.push(this.genLable(steps[i], l))

          // 处理每个 node 的数据
          nodes.forEach(node => {
            const nodeObj = obj.filter(o => o.node === node)

            // 上传数据 (direction = true)
            const upTraffics = nodeObj.filter(o => o.direction && o.dateTime * 1000 < steps[i] && o.dateTime * 1000 > steps[i - 1]).map((o: any) => o.traffic)
            const upSum = upTraffics.length > 0 ? upTraffics.reduce((u: number, v: number) => u + v, 0) : null

            // 下载数据 (direction = false)
            const downTraffics = nodeObj.filter(o => !o.direction && o.dateTime * 1000 < steps[i] && o.dateTime * 1000 > steps[i - 1]).map((o: any) => o.traffic)
            const downSum = downTraffics.length > 0 ? downTraffics.reduce((d: number, v: number) => d + v, 0) : null

            nodeData.get(node)!.uplink.push(upSum)
            nodeData.get(node)!.downlink.push(downSum)
          })
        }

        // 构建 datasets
        const datasets: any[] = []
        nodes.forEach((node, index) => {
          const data = nodeData.get(node)!

          // 上传数据集
          datasets.push({
            label: `${node} - ${i18n.global.t('stats.upload')}`,
            backgroundColor: this.getNodeColor(node, index, true, true),
            borderColor: this.getNodeColor(node, index, true, false),
            fill: true,
            data: data.uplink
          })

          // 下载数据集
          datasets.push({
            label: `${node} - ${i18n.global.t('stats.download')}`,
            backgroundColor: this.getNodeColor(node, index, false, true),
            borderColor: this.getNodeColor(node, index, false, false),
            fill: true,
            data: data.downlink
          })
        })

        this.usage = {
          labels: labels,
          datasets: datasets
        }
        this.loaded = true
        this.alert = false
      } else {
        this.alert = true
        this.loaded = false
      }
      this.loading = false
    },
    genLable(step:number, locale: string) {
      return new Date(step).toLocaleString(locale,{
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit',
        hour12: false,
      })
    },
  },
  watch: {
    visible(v) {
      if (v) {
        this.limit = 1
        this.loadData()
        this.intervalId = setInterval(() => {
          this.loadData()
        }, 10000)
      } else {
        this.loaded = false
        this.alert = false
        this.usage.labels = []
        if (this.usage.datasets) {
          this.usage.datasets[0].data = []
          this.usage.datasets[1].data = []
        }
        if (this.intervalId && this.intervalId != 0) {
          clearInterval(this.intervalId)
        }
      }
    }
  }
}
</script>
