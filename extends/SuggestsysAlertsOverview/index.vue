<template>
  <div class="h-100 position-relative">
    <div class="dashboard-card-wrap">
      <div class="dashboard-card-header">
        <div class="dashboard-card-header-left">{{ fd.name }}<a-icon class="ml-2" type="loading" v-if="loading" /></div>
        <div class="dashboard-card-header-right">
          <slot name="actions" :handle-edit="() => visible = true" />
          <router-link v-if="!edit" to="/suggestsysalert" class="ml-2">{{$t('dashboard.more')}}</router-link>
        </div>
      </div>
      <div class="dashboard-card-body align-items-center justify-content-center flex-column">
        <div class="flex-shrink-0 flex-grow-0 font-weight-bold mt-2">{{$t('dashboard.text_50')}} <span class="success-color">{{ percent }}%</span> {{$t('dashboard.text_51')}}</div>
        <div class="flex-fill w-100">
          <e-chart :options="chartOptions" style="height: 100%; width: 100%;" autoresize />
        </div>
        <div class="w-100">
          <div class="d-flex">
            <div class="flex-grow-0 flex-shrink-0 text-color-help">{{$t('dashboard.text_52')}}</div>
            <div class="flex-fill text-right error-color font-weight-bold">{{ forcastAmountFormat }}</div>
          </div>
          <div class="d-flex mt-2">
            <div class="flex-grow-0 flex-shrink-0 text-color-help">{{$t('dashboard.text_53')}}</div>
            <div class="flex-fill text-right success-color font-weight-bold">{{ suggestAmountFormat }}</div>
          </div>
        </div>
      </div>
    </div>
    <base-drawer :visible.sync="visible" :title="$t('dashboard.text_5')" @ok="handleSubmit">
      <a-form-model
        ref="form"
        hideRequiredMark
        :model="fd"
        :rules="rules">
        <a-form-model-item :label="$t('dashboard.text_6')" prop="name">
          <a-input v-model="fd.name" />
        </a-form-model-item>
        <a-form-model-item :label="$t('dashboard.currency')" prop="currency">
          <a-select v-model="fd.currency">
            <a-select-option v-for="obj in newCurrencys" :key="obj.key" :value="obj.key">
              {{ obj.value }}
            </a-select-option>
          </a-select>
        </a-form-model-item>
      </a-form-model>
    </base-drawer>
  </div>
</template>

<script>
import { mapGetters, mapState } from 'vuex'
import BaseDrawer from '@Dashboard/components/BaseDrawer'
import { load } from '@Dashboard/utils/cache'
import { getRequestT } from '@/utils/utils'
import { getCurrency } from '@/utils/common/cookie'
import { CURRENCYS } from '@/constants'

export default {
  name: 'SuggestsysAlertsOverview',
  components: {
    BaseDrawer,
  },
  props: {
    options: {
      type: Object,
      required: true,
    },
    params: Object,
    edit: Boolean,
  },
  data () {
    const initNameValue = (this.params && this.params.name) || this.$t('dashboard.text_54')
    const initCurrencyValue = (this.params && this.params.currency) || getCurrency()
    return {
      CURRENCYS,
      data: {},
      visible: false,
      loading: false,
      fd: {
        name: initNameValue,
        currency: initCurrencyValue,
      },
      rules: {
        name: [
          { required: true, message: this.$t('dashboard.text_8') },
        ],
      },
    }
  },
  computed: {
    ...mapState('common', {
      currency: state => state.bill.currency,
      currencyOpts: state => state.bill.currencyOpts,
    }),
    ...mapGetters(['scope']),
    forcastAmount () {
      return this.data.meter_forcast_cost && this.data.meter_forcast_cost.amount
    },
    suggestAmount () {
      return this.data.suggest_cost && this.data.suggest_cost[0] && this.data.suggest_cost[0].amount
    },
    currencySign () {
      return this.fd.currency === 'USD' ? '$' : '¥'
    },
    forcastAmountFormat () {
      if (!this.forcastAmount) {
        return `${this.currencySign}0`
      }
      return `${this.currencySign}${this.forcastAmount.toFixed(2)}`
    },
    suggestAmountFormat () {
      if (!this.suggestAmount) {
        return `${this.currencySign}0`
      }
      return `${this.currencySign}${this.suggestAmount.toFixed(2)}`
    },
    percent () {
      let percent = ((this.suggestAmount / this.forcastAmount) || 0) * 100
      if (percent && percent < 10) {
        return percent.toFixed(1)
      }
      percent = parseInt(percent)
      if (isNaN(percent)) return 0
      return percent
    },
    chartOptions () {
      return {
        title: {
          text: `${this.percent}%`,
          x: 'center',
          y: 'center',
          textStyle: {
            fontWeight: 'normal',
            color: 'rgb(150, 150, 150)',
            fontSize: '20',
          },
        },
        color: ['rgba(176, 212, 251, 1)'],
        legend: {
          show: false,
          itemGap: 12,
          data: ['forcastAmount', 'suggestAmount'],
        },
        series: [{
          name: 'Line 1',
          type: 'pie',
          clockWise: true,
          radius: ['50%', '66%'],
          itemStyle: {
            normal: {
              label: {
                show: false,
              },
              labelLine: {
                show: false,
              },
            },
          },
          hoverAnimation: false,
          data: [{
            value: this.forcastAmount,
            name: 'forcastAmount',
            itemStyle: {
              normal: {
                color: '#1890ff',
                label: {
                  show: false,
                },
                labelLine: {
                  show: false,
                },
              },
            },
          }, {
            name: 'suggestAmount',
            value: this.suggestAmount,
            itemStyle: {
              normal: {
                color: '#52c41a',
                label: {
                  show: false,
                },
                labelLine: {
                  show: false,
                },
              },
            },
          }],
        }],
      }
    },
    newCurrencys () {
      return this.CURRENCYS.filter(v => {
        return this.currencyOpts.find(obj => obj.item_id === v.key)
      })
    },
  },
  watch: {
    'fd.currency' (val) {
      this.fetchData()
    },
  },
  created () {
    if (this.params && !this.params.currency) {
      this.fd.currency = this.currency
    }
    this.fetchData()
    this.$emit('update', this.options.i, {
      ...this.fd,
    })
  },
  methods: {
    refresh () {
      return this.fetchData()
    },
    async fetchData () {
      this.loading = true
      try {
        const data = await load({
          res: 'quotas',
          actionArgs: {
            url: '/v1/suggestsysalerts/cost',
            method: 'GET',
            params: {
              $t: getRequestT(),
              details: false,
              scope: this.scope,
              currency: this.fd.currency,
            },
          },
          useManager: false,
          resPath: 'data',
        })
        this.data = data || {}
      } finally {
        this.loading = false
      }
    },
    async handleSubmit () {
      try {
        await this.$refs.form.validate()
        this.$emit('update', this.options.i, this.fd)
        this.visible = false
      } catch (error) {
        throw error
      }
    },
  },
}
</script>
