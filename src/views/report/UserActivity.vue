<!-- Report UserActivity tab -->
<template>
  <div>
    <a-form layout="inline" :form="form" >
      <om-report-date-picker @ok="handleApply"/>
      <om-pub-app-select :defaultValue="def.defaultApps" @change="appSelect"/>
      <a-form-item>
        <om-regions-select :defaultSelected="def.defaultRegions" @change="countryChange" :ignoreApp="true" style="width: 220px"/>
      </a-form-item>
      <om-placement-select-simple :pubAppIds="pubAppIds" :defaultValue="def.defaultPlacements" @change="placementSelect" :form="form" width="225px"/>
      <a-form-item>
        <a-button ghost @click="handleApply" type="primary" style="width: 84px;">Apply</a-button>
      </a-form-item>
    </a-form>

    <div style="background-color: white; margin-top: 8px; padding: 16px">
      <div style="margin-bottom: 16px;">
        <span style="font-size: 16px;color: #333333;font-weight:500;margin-right: 4px;">User Activity by</span>
        <a-dropdown :trigger="['click']">
          <a class="ant-dropdown-link" style="font-size:16px;font-weight:bold;" @click="e => e.preventDefault()">
            {{ chartGroupBreakdown }} <a-icon type="down" />
          </a>
          <a-menu slot="overlay">
            <a-menu-item
              v-for="s in dimList4Chart"
              :key="s.id"
              :title="s.title"
              v-if="['pubAppId','country'].includes(s.id) || (['placementId'].includes(s.id) && placementAllow)|| (['adnId'].includes(s.id) && adnAllow) || (['instanceId'].includes(s.id) && instanceAllow)" >
              <a href="javascript:;" @click="handleDim4ChartChange(s)">{{ s.title }}</a>
            </a-menu-item>
          </a-menu>
        </a-dropdown>
      </div>

      <a-row :gutter="16">
        <a-col :span="8">
          <om-report-chart
            title="DAU"
            :loading="loading"
            :data="chartData"
            :height="248"
            x-column="day"
            y-column="dau"
            y-format="0,0.[00]a"
            :group-by="chartGroupViewDim" />
        </a-col>
        <a-col :span="8">
          <om-report-chart
            title="ARPDAU"
            :loading="loading"
            :data="chartData"
            :height="248"
            x-column="day"
            y-column="arpDau"
            y-format="$0.[0000]"
            :group-by="chartGroupViewDim" />
        </a-col>
        <a-col :span="8">
          <om-report-chart
            title="Impression per DAU"
            :loading="loading"
            :data="chartData"
            :height="248"
            x-column="day"
            y-column="imprDau"
            :group-by="chartGroupViewDim" />
        </a-col>
      </a-row>
      <a-row :gutter="16" style="margin-top: 16px;">
        <a-col :span="8">
          <om-report-chart
            title="DEU"
            :loading="loading"
            :data="chartData"
            :height="248"
            x-column="day"
            y-column="deu"
            y-format="0,0.[00]a"
            :group-by="chartGroupViewDim" />
        </a-col>
        <a-col :span="8">
          <om-report-chart
            title="ARPDEU"
            :loading="loading"
            :data="chartData"
            :height="248"
            x-column="day"
            y-column="arpDeu"
            y-format="$0.[0000]"
            :group-by="chartGroupViewDim" />
        </a-col>
        <a-col :span="8">
          <om-report-chart
            title="Impression per DEU"
            :loading="loading"
            :data="chartData"
            :height="248"
            x-column="day"
            y-column="imprDeu"
            :group-by="chartGroupViewDim" />
        </a-col>
      </a-row>
    </div>

    <div style="background-color: white; margin-top: 16px; padding: 16px">
      <div style="margin-bottom: 4px;color: #999999;">
        <div style="display: inline-block;">Breakdown</div>
        <div style="position: absolute; margin-top: -20px; margin-left: 258px;">Metrics</div>
      </div>
      <a-select
        showSearch
        style="width:240px;margin-bottom:16px;"
        placeholder="Breakdown"
        mode="multiple"
        :maxTagCount="1"
        :allowClear="true"
        optionLabelProp="title"
        @change="handleDim4TableChange"
        :value="table.dimension">
        <a-select-option v-for="o in dimList4Table" :key="o.id" :title="o.title">{{ o.title }}</a-select-option>
      </a-select>

      <a-select
        showSearch
        style="width:304px;margin:0 0 16px 16px;"
        placeholder="Metrics"
        mode="multiple"
        :maxTagCount="1"
        :allowClear="true"
        :showArrow="false"
        optionLabelProp="title"
        @change="handleMetricsChange"
        :value="table.metric">
        <a-select-option v-for="(o, key) in supportedMetrics" :key="key" :title="o.title">{{ o.title }}</a-select-option>
      </a-select>

      <a @click="handleDownload" style="float: right; margin-right: 16px" :disabled="table.data.length===0">CSV</a>
      <a-table
        v-if="table.data.length"
        class="ant-card-table-default"
        :scroll="{x: true}"
        :loading="table.loading"
        :dataSource="table.data"
        :columns="table.columns"
        @change="tableChange"
        row-key="id"
        :pagination="false"/>
      <div v-else class="performance_no_data">
        <div class="no_data">
          <img src="/icon/empty.svg" />
        </div>
      </div>
    </div>
    <ul v-if="table.data.length" style="list-style:disc;margin-left: -16px;margin-top:8px;color: #999999;">
      <li>All reports and dates are in UTC</li>
    </ul>
  </div>
</template>

<script>
import { Ellipsis } from '../../components'
import { OmRegionsSelect, OmReportDatePicker, OmPubAppSelect, OmPlacementSelectSimple } from '../../components/om'
import OmReportChart from './ChartCardG2.js'
import { getReportList } from '@/api/report'
import { generateUUID } from 'ant-design-vue/lib/vc-select/util'
import numerify from 'numerify'
import numerifyCurrency from 'numerify/lib/plugins/currency.es'
import numerifyPercent from 'numerify/lib/plugins/percent.es'
import { mapState } from 'vuex'
numerify.register('currency', numerifyCurrency)
numerify.register('percent', numerifyPercent)

const supportedDimensions = {
  day: 'Day',
  country: 'Regions',
  pubAppId: 'Apps',
  placementId: 'Placements',
  adnId: 'Ad Network',
  instanceId: 'Instances'
}

const dimColumnMapper = {
  pubAppId: 'pubAppName',
  placementId: 'placementName',
  adnId: 'adnName',
  instanceId: 'instanceName'
}

export default {
  name: 'OmUserActivityReport',
  components: {
    OmPubAppSelect,
    OmReportChart,
    OmReportDatePicker,
    OmRegionsSelect,
    Ellipsis,
    OmPlacementSelectSimple
  },
  data () {
    const def = {
      defaultApps: [],
      defaultRegions: [],
      defaultPlacements: [],
      dimension: ['day'],
      metric: ['dau', 'deu']
    }
    let lastCondition = localStorage.getItem('condition-ua-' + this.$store.state.publisher.currentOrgId)
    if (lastCondition) {
      lastCondition = JSON.parse(lastCondition)
      const { pubAppId = [], country = [], dimension = ['day'], metric = ['dau', 'deu'] } = { ...lastCondition }
      def.defaultApps = pubAppId
      this.appSelect(pubAppId)
      def.defaultRegions = country
      def.dimension = dimension
      def.metric = metric
    }
    const queryPubAppId = this.$route.query.pubAppId
    if (queryPubAppId) {
      def.defaultApps = [queryPubAppId]
    }
    return {
      def,
      form: this.$form.createForm(this),
      regions: def.defaultRegions || [],
      country: [],
      placementAllow: false,
      adnAllow: false,
      instanceAllow: false,
      placementIds: [],
      supportedMetrics: {
        dau: { title: 'DAU' },
        arpDau: { title: 'ARPDAU', format: '$0.[0000]' },
        imprDau: { title: 'Impressions / DAU', format: '0.00' },
        deu: { title: 'DEU' },
        arpDeu: { title: 'ARPDEU', format: '$0.[0000]' },
        imprDeu: { title: 'Impressions / DEU', format: '0.00' },
        engagementRate: { title: 'Engagement Rate', format: '0.00 %' }
      },
      dimList4Chart: this.filterDim('pubAppId', 'country', 'placementId', 'adnId', 'instanceId'),
      dimList4Table: this.filterDim('day', 'country', 'pubAppId', 'placementId', 'adnId', 'instanceId'),
      loading: false,
      chartGroupBy: 'pubAppId',
      chartData: [],
      pubAppIds: def.defaultApps,
      sorter: null,
      table: {
        loading: false,
        dimension: def.dimension,
        metric: def.metric,
        data: [],
        columns: []
      }
    }
  },
  computed: {
    chartGroupViewDim () {
      return dimColumnMapper[this.chartGroupBy] || this.chartGroupBy
    },
    ...mapState({
      currentOrgId: state => state.publisher.currentOrgId
    }),
    chartGroupBreakdown () {
      return supportedDimensions[this.chartGroupBy] || this.chartGroupBy
    }
  },
  mounted () {
    if (this.pubAppIds && this.pubAppIds.length === 1) {
      if ((this.placementIds && this.placementIds.length === 1) || !this.placementIds || !this.placementIds.length) {
        this.adnAllow = true
      }
      this.placementAllow = true
    }
    if (this.placementIds && this.placementIds.length === 1) {
      this.instanceAllow = true
    }
    this.reloadChart()
    this.reloadTable('dimension')
  },
  methods: {
    appSelect (val) {
      this.pubAppIds = val
      if (val.length === 1) {
        if ((this.placementIds && this.placementIds.length === 1) || !this.placementIds || !this.placementIds.length) {
          this.adnAllow = true
        }
        this.placementAllow = true
      } else {
        this.adnAllow = false
        this.placementAllow = false
        if (this.chartGroupBy === 'placementId') {
          this.chartGroupBy = 'pubAppId'
          this.reloadChart()
        }
      }
    },
    placementSelect (val) {
      this.placementIds = val
      if (val.length === 1) {
        this.instanceAllow = true
        if (this.pubAppIds && this.pubAppIds.length === 1) {
          this.adnAllow = true
        }
      } else if (val.length === 0) {
        this.instanceAllow = false
        if (this.pubAppIds && this.pubAppIds.length === 1) {
          this.adnAllow = true
        }
      } else {
        this.instanceAllow = false
        this.adnAllow = false
        if (this.chartGroupBy === 'instanceId') {
          this.chartGroupBy = 'pubAppId'
          this.reloadChart()
        }
      }
    },
    filterDim (...ids) {
      return ids.map(id => {
        return { id, title: supportedDimensions[id] }
      })
    },
    countryChange (val) {
      this.regions = val
    },
    handleApply () {
      this.reloadChart()
      this.reloadTable('dimension')
    },
    handleDim4ChartChange (dim) {
      this.chartGroupBy = dim.id
      this.reloadChart()
    },
    handleDim4TableChange (dim) {
      this.table.dimension = dim
      this.reloadTable('dimension')
    },
    handleMetricsChange (val) {
      this.table.metric = val
      this.reloadTable('metric')
    },
    buildBaseParams () {
      const ps = this.form.getFieldsValue()
      ps.dateBegin = ps.dateRange[0].format('YYYY-MM-DD')
      ps.dateEnd = ps.dateRange[1].format('YYYY-MM-DD')
      delete ps.dateRange
      ps.country = JSON.parse(JSON.stringify(this.regions))
      if (ps.country && ps.country.indexOf('ALL') > -1) {
        ps.country.splice(ps.country.indexOf('ALL'), 1)
        ps.country.push('00')
      }
      ps.type = ['dau', 'api']
      ps.dimension = ['day'].concat(this.chartGroupBy)
      return ps
    },
    reloadChart () {
      const ps = this.buildBaseParams()
      this.loading = true
      getReportList(ps)
        .then(res => {
          if (!res.code) {
            this.chartData = res.data.map(this.mapMetrics).sort((a, b) => b.cost || 0 - a.cost || 0)
          }
        })
        .finally(_ => {
          this.loading = false
        })
    },
    reloadTable (triggerType) {
      const { dimension, metric } = this.table
      if (dimension.length === 0 || metric.length === 0) {
        this.table.columns = []
        this.table.data = []
        return
      }
      const ps = this.buildBaseParams()
      const { pubAppId, country } = { ...ps }
      ps.dimension = dimension
      ps.metric = metric
      const cond = { pubAppId, country, dimension, metric }
      localStorage.setItem('condition-ua-' + this.currentOrgId, JSON.stringify(cond))
      const columns = dimension.map(dim => {
        const di = dimColumnMapper[dim] || dim
        return {
          title: supportedDimensions[dim],
          dataIndex: di,
          customHeaderCell: (custom) => {
            return {
              style: { borderBottom: 'none' },
              class: 'adt-cell-blue'
            }
          },
          sorter: (a, b) => a[di].localeCompare(b[di])
        }
      })
      metric.forEach(name => {
        const metric = this.supportedMetrics[name]
        if (!metric.format) {
          metric.format = '0,0'
        }
        columns.push({
          title: metric.title,
          dataIndex: name,
          customRender (v, row, i) {
            return numerify(v, metric.format)
          },
          customHeaderCell: (custom) => {
            return {
              style: { borderBottom: 'none' },
              class: 'adt-cell-green'
            }
          },
          align: 'right',
          sorter (a, b) {
            return a[name] - b[name]
          }
        })
      })
      this.table.columns = columns

      if (this.table.data.length === 0 || triggerType === 'dimension') {
        this.table.loading = true
        getReportList(ps)
          .then(res => {
            if (!res.code) {
              this.table.data = res.data.map(this.mapMetrics).sort((a, b) => {
                if (b.day) {
                  return b.day.localeCompare(a.day)
                }
                return 0
              })
            }
          })
          .finally(_ => {
            this.table.loading = false
          })
      }
    },
    mapMetrics (row) {
      row = {
        id: generateUUID(),
        apiImpr: 0,
        cost: 0,
        deu: 0,
        dau: 0,
        ...row
      }
      row.impr = row.apiImpr
      row.arpDau = row.arpDeu = row.imprDau = row.imprDeu = row.engagementRate = 0

      if (row.dau > 0) {
        row.arpDau = row.cost / row.dau
        row.imprDau = row.impr / row.dau
        row.engagementRate = row.deu / row.dau
      }
      if (row.deu > 0) {
        row.arpDeu = row.cost / row.deu
        row.imprDeu = row.impr / row.deu
      }
      if (!row.country) {
        row.country = '(Empty)'
      }
      if (!row.pubAppName) {
        row.pubAppName = '(Empty)'
      }
      if (!row.placementName) {
        row.placementName = '(Empty)'
      }
      if (!row.instanceName) {
        row.instanceName = '(Empty)'
      }
      if (!row.adnName) {
        row.adnName = '(Empty)'
      }
      return row
    },
    tableChange (pagination, filters, sorter) {
      this.sorter = sorter
    },
    handleDownload () {
      const header = this.table.columns.map(col => col.title).join(',')
      let data = [...this.table.data]
      const that = this
      if (this.sorter) {
        data = data.sort((a, b) => {
          if (that.sorter.order === 'ascend') {
            return a[that.sorter.field].localeCompare ? a[that.sorter.field].localeCompare(b[that.sorter.field]) : a[that.sorter.field] - b[that.sorter.field]
          } else {
            return b[that.sorter.field].localeCompare ? b[that.sorter.field].localeCompare(a[that.sorter.field]) : b[that.sorter.field] - a[that.sorter.field]
          }
        })
      }
      data = data.map(row => {
        return this.table.columns.map(col => {
          return row[col.dataIndex]
        }).join(',')
      }).join('\n')
      const blob = new Blob([new Uint8Array([0xef, 0xbb, 0xbf]), header + '\n' + data], { type: 'text/csv;charset=UTF-8' })
      const link = document.createElement('a')
      link.href = URL.createObjectURL(blob)
      link.download = 'download.csv'
      link.click()
      URL.revokeObjectURL(link.href)
    }
  }
}
</script>
<style lang="less" scoped>
  .ant-form-item {
    margin-right: 8px;
    button {
      width: 100px;
    }
  }
  .performance_no_data {
    background: #FBFBFB;
    height: 176px;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .performance_no_data div > div {
    font-weight: bold;
    font-size: 16px;
    line-height: 18px;
    text-align: center;
  }
</style>
