<template>
  <data-view :title="title" :title-id="titleId" :date="date">
    <template v-slot:attentionNote>
      <slot name="attentionNote" />
    </template>
    <template v-slot:description>
      <slot name="description" />
    </template>
    <template v-slot:button>
      <data-selector
        v-model="dataKind"
        :target-id="chartId"
        :style="{ display: canvas ? 'inline-block' : 'none' }"
      />
    </template>
    <h4 :id="`${titleId}-graph`" class="visually-hidden">
      {{ $t(`{title}のグラフ`, { title }) }}
    </h4>
    <scrollable-chart v-show="canvas" :display-data="displayData">
      <template v-slot:chart="{ chartWidth }">
        <bar
          :ref="'barChart'"
          :chart-id="chartId"
          :chart-data="displayData"
          :options="displayOption"
          :height="240"
          :width="chartWidth"
        />
      </template>
      <template v-slot:sticky-chart>
        <bar
          class="sticky-legend"
          :chart-id="`${chartId}-header`"
          :chart-data="displayDataHeader"
          :options="displayOptionHeader"
          :plugins="yAxesBgPlugin"
          :height="240"
        />
      </template>
    </scrollable-chart>
    <template v-slot:additionalDescription>
      <slot name="additionalDescription" />
    </template>
    <template v-slot:dataTable>
      <data-view-table :headers="tableHeaders" :items="tableData" />
    </template>
    <template v-slot:infoPanel>
      <data-view-basic-info-panel
        :l-text="displayInfo.lText"
        :s-text="displayInfo.sText"
        :unit="displayInfo.unit"
      />
    </template>
    <template v-slot:footer>
      <open-data-link v-show="url" :url="url" />
    </template>
  </data-view>
</template>

<script lang="ts">
import Vue from 'vue'
import { ThisTypedComponentOptionsWithRecordProps } from 'vue/types/options'
import { Chart } from 'chart.js'
import dayjs from 'dayjs'
import { GraphDataType } from '@/utils/formatGraph'
import DataView from '@/components/DataView.vue'
import DataSelector from '@/components/DataSelector.vue'
import DataViewBasicInfoPanel from '@/components/DataViewBasicInfoPanel.vue'
import DataViewTable, {
  TableHeader,
  TableItem
} from '@/components/DataViewTable.vue'
import ScrollableChart from '@/components/ScrollableChart.vue'
import OpenDataLink from '@/components/OpenDataLink.vue'
import { DisplayData, yAxesBgPlugin } from '@/plugins/vue-chart'

import { getGraphSeriesStyle } from '@/utils/colors'
import { getComplementedDate } from '@/utils/formatDate'

type Data = {
  dataKind: 'transition' | 'cumulative'
  canvas: boolean
}
type Methods = {
  formatDayBeforeRatio: (dayBeforeRatio: number) => string
}

type Computed = {
  displayCumulativeRatio: string
  displayTransitionRatio: string
  displayInfo: {
    lText: string
    sText: string
    unit: string
  }
  displayData: DisplayData
  displayOption: Chart.ChartOptions
  displayDataHeader: DisplayData
  displayOptionHeader: Chart.ChartOptions
  scaledTicksYAxisMax: number
  tableHeaders: TableHeader[]
  tableData: TableItem[]
}
type Props = {
  title: string
  titleId: string
  chartId: string
  chartData: GraphDataType[]
  date: string
  unit: string
  url: string
  yAxesBgPlugin: Chart.PluginServiceRegistrationOptions[]
  byDate: boolean
}

const options: ThisTypedComponentOptionsWithRecordProps<
  Vue,
  Data,
  Methods,
  Computed,
  Props
> = {
  created() {
    this.canvas = process.browser
    this.dataKind =
      this.$route.query.embed && this.$route.query.dataKind === 'cumulative'
        ? 'cumulative'
        : 'transition'
  },
  components: {
    DataView,
    DataSelector,
    DataViewBasicInfoPanel,
    DataViewTable,
    ScrollableChart,
    OpenDataLink
  },
  props: {
    title: {
      type: String,
      default: ''
    },
    titleId: {
      type: String,
      default: ''
    },
    chartId: {
      type: String,
      default: 'time-bar-chart'
    },
    chartData: {
      type: Array,
      default: () => []
    },
    date: {
      type: String,
      required: true
    },
    unit: {
      type: String,
      default: ''
    },
    url: {
      type: String,
      default: ''
    },
    yAxesBgPlugin: {
      type: Array,
      default: () => yAxesBgPlugin
    },
    byDate: {
      type: Boolean,
      default: false
    }
  },
  data: () => ({
    dataKind: 'transition',
    canvas: true
  }),
  computed: {
    displayCumulativeRatio() {
      const lastDay = this.chartData.slice(-1)[0].cumulative
      const lastDayBefore = this.chartData.slice(-2)[0].cumulative
      return this.formatDayBeforeRatio(lastDay - lastDayBefore)
    },
    displayTransitionRatio() {
      const lastDay = this.chartData.slice(-1)[0].transition
      const lastDayBefore = this.chartData.slice(-2)[0].transition
      return this.formatDayBeforeRatio(lastDay - lastDayBefore)
    },
    displayInfo() {
      const date = this.chartData.slice(-1)[0].label
      if (this.dataKind === 'transition' && this.byDate) {
        return {
          lText: `${this.chartData.slice(-1)[0].transition.toLocaleString()}`,
          sText: `${date} ${this.$t('日別値')}（${this.$t('前日比')}: ${
            this.displayTransitionRatio
          } ${this.unit}）`,
          unit: this.unit
        }
      } else if (this.dataKind === 'transition') {
        return {
          lText: `${this.chartData.slice(-1)[0].transition.toLocaleString()}`,
          sText: `${date} ${this.$t('実績値')}（${this.$t('前日比')}: ${
            this.displayTransitionRatio
          } ${this.unit}）`,
          unit: this.unit
        }
      }
      return {
        lText: this.chartData[
          this.chartData.length - 1
        ].cumulative.toLocaleString(),
        sText: `${date} ${this.$t('累計値')}（${this.$t('前日比')}: ${
          this.displayCumulativeRatio
        } ${this.unit}）`,
        unit: this.unit
      }
    },
    displayData() {
      const style = getGraphSeriesStyle(0)[0]
      const zeroMouseOverHeight = 5
      const transparentWhite = 'rgba(255,255,255,0)'

      if (this.dataKind === 'transition') {
        return {
          labels: this.chartData.map(d => {
            return d.label
          }),
          datasets: [
            {
              label: this.dataKind,
              data: this.chartData.map(_d => {
                return 0
              }),
              backgroundColor: transparentWhite,
              borderColor: transparentWhite,
              borderWidth: 0,
              minBarLength: this.chartData.map(d => {
                if (d.transition <= 0) {
                  return zeroMouseOverHeight
                }
                return 0
              })
            },
            {
              label: this.dataKind,
              data: this.chartData.map(d => {
                return d.transition
              }),
              backgroundColor: style.fillColor,
              borderColor: style.strokeColor,
              borderWidth: 1
            }
          ]
        }
      }
      return {
        labels: this.chartData.map(d => d.label),
        datasets: [
          {
            label: this.dataKind,
            data: this.chartData.map(_d => {
              return 0
            }),
            backgroundColor: transparentWhite,
            borderColor: transparentWhite,
            borderWidth: 0,
            minBarLength: this.chartData.map(d => {
              if (d.cumulative <= 0) {
                return zeroMouseOverHeight
              }
              return 0
            })
          },
          {
            label: this.dataKind,
            data: this.chartData.map(d => {
              return d.cumulative
            }),
            backgroundColor: style.fillColor,
            borderColor: style.strokeColor,
            borderWidth: 1
          }
        ]
      }
    },
    displayOption() {
      const self = this
      const unit = this.unit
      const scaledTicksYAxisMax = this.scaledTicksYAxisMax
      const options: Chart.ChartOptions = {
        tooltips: {
          displayColors: false,
          callbacks: {
            label(tooltipItem) {
              const labelText = `${parseInt(
                tooltipItem.value!
              ).toLocaleString()} ${unit}`
              return labelText
            },
            title(tooltipItem, data) {
              const label = data.labels![tooltipItem[0].index!] as string
              return self.$d(
                new Date(getComplementedDate(label)),
                'dateWithoutYear'
              )
            }
          }
        },
        maintainAspectRatio: false,
        legend: {
          display: false
        },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: '#808080',
                maxRotation: 0,
                callback: (label: string) => {
                  return label.split('/')[1]
                }
              }
              // #2384: If you set "type" to "time", make sure that the bars at both ends are not hidden.
              // #2384: typeをtimeに設定する時はグラフの両端が見切れないか確認してください
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: true,
                drawBorder: false,
                tickMarkLength: 10
              },
              ticks: {
                fontSize: 11,
                fontColor: '#808080',
                padding: 3,
                fontStyle: 'bold'
              },
              type: 'time',
              time: {
                unit: 'month',
                parser: 'M/D',
                displayFormats: {
                  month: 'MMM'
                }
              }
            }
          ],
          yAxes: [
            {
              stacked: true,
              gridLines: {
                display: true,
                color: '#E5E5E5'
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080',
                suggestedMax: scaledTicksYAxisMax
              }
            }
          ]
        }
      }
      if (this.$route.query.ogp === 'true') {
        Object.assign(options, { animation: { duration: 0 } })
      }
      return options
    },
    displayDataHeader() {
      if (this.dataKind === 'transition') {
        return {
          labels: ['2020/1/1'],
          datasets: [
            {
              data: [Math.max(...this.chartData.map(d => d.transition))],
              backgroundColor: 'transparent',
              borderWidth: 0
            }
          ]
        }
      }
      return {
        labels: ['2020/1/1'],
        datasets: [
          {
            data: [Math.max(...this.chartData.map(d => d.cumulative))],
            backgroundColor: 'transparent',
            borderWidth: 0
          }
        ]
      }
    },
    displayOptionHeader() {
      const options: Chart.ChartOptions = {
        maintainAspectRatio: false,
        legend: {
          display: false
        },
        tooltips: { enabled: false },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: 'transparent',
                maxRotation: 0,
                minRotation: 0,
                callback: (label: string) => {
                  return label.split('/')[1]
                }
              }
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: false, // true -> false
                drawBorder: false,
                tickMarkLength: 10
              },
              ticks: {
                fontSize: 11,
                fontColor: 'transparent', // #808080
                padding: 13, // 3 + 10(tickMarkLength)
                fontStyle: 'bold',
                callback: (label: string) => {
                  const monthStringArry = [
                    'Jan',
                    'Feb',
                    'Mar',
                    'Apr',
                    'May',
                    'Jun',
                    'Jul',
                    'Aug',
                    'Sep',
                    'Oct',
                    'Nov',
                    'Dec'
                  ]
                  const month = monthStringArry.indexOf(label.split(' ')[0]) + 1
                  return month + '月'
                }
              },
              type: 'time',
              time: {
                unit: 'month'
              }
            }
          ],
          yAxes: [
            {
              stacked: true,
              gridLines: {
                display: true,
                drawOnChartArea: false,
                color: '#E5E5E5' // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080' // #808080
              }
            }
          ]
        },
        animation: { duration: 0 }
      }
      return options
    },
    scaledTicksYAxisMax() {
      const dataKind =
        this.dataKind === 'transition' ? 'transition' : 'cumulative'
      const values = this.chartData.map(d => d[dataKind])
      return Math.max(...values)
    },
    tableHeaders() {
      return [
        { text: this.$t('日付'), value: 'text' },
        {
          text: `${this.title} (${this.$t('日別')})`,
          value: 'transition',
          align: 'end'
        },
        {
          text: `${this.title} (${this.$t('累計')})`,
          value: 'cumulative',
          align: 'end'
        }
      ]
    },
    tableData() {
      return this.chartData
        .map((d, _) => {
          return {
            text: this.$d(
              new Date(getComplementedDate(d.label)),
              'dateWithoutYear'
            ),
            transition: d.transition.toLocaleString(),
            cumulative: d.cumulative.toLocaleString()
          }
        })
        .sort((a, b) => dayjs(a.text).unix() - dayjs(b.text).unix())
        .reverse()
    }
  },
  methods: {
    formatDayBeforeRatio(dayBeforeRatio: number): string {
      const dayBeforeRatioLocaleString = dayBeforeRatio.toLocaleString()
      switch (Math.sign(dayBeforeRatio)) {
        case 1:
          return `+${dayBeforeRatioLocaleString}`
        case -1:
          return `${dayBeforeRatioLocaleString}`
        default:
          return `${dayBeforeRatioLocaleString}`
      }
    }
  },
  mounted() {
    const barChart = this.$refs.barChart as Vue
    const barElement = barChart.$el
    const canvas = barElement.querySelector('canvas')
    const labelledbyId = `${this.titleId}-graph`

    if (canvas) {
      canvas.setAttribute('role', 'img')
      canvas.setAttribute('aria-labelledby', labelledbyId)
    }
  }
}

export default Vue.extend(options)
</script>
