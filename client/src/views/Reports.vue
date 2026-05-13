<template>
  <div class="reports">
    <div class="page-header">
      <h2>{{ t('reports.title') }}</h2>
      <p>{{ t('reports.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Quarterly Performance -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.quarterlyTitle') }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t('reports.table.quarter') }}</th>
                <th>{{ t('reports.table.totalOrders') }}</th>
                <th>{{ t('reports.table.totalRevenue') }}</th>
                <th>{{ t('reports.table.avgOrderValue') }}</th>
                <th>{{ t('reports.table.fulfillmentRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="q in quarterlyData" :key="q.quarter">
                <td><strong>{{ q.quarter }}</strong></td>
                <td>{{ q.total_orders.toLocaleString(numberLocale) }}</td>
                <td>{{ formatCurrency(q.total_revenue, currentCurrency) }}</td>
                <td>{{ formatCurrency(q.avg_order_value, currentCurrency) }}</td>
                <td>
                  <span :class="getFulfillmentClass(q.fulfillment_rate)">
                    {{ q.fulfillment_rate }}%
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Monthly Revenue Trend Chart -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monthlyTitle') }}</h3>
        </div>
        <div class="chart-container">
          <div class="bar-chart">
            <div v-for="bar in monthlyBars" :key="bar.month" class="bar-wrapper">
              <div class="bar-container">
                <div
                  class="bar"
                  :style="{ height: bar.height + 'px' }"
                  :title="bar.label"
                ></div>
              </div>
              <div class="bar-label">{{ bar.monthLabel }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Month-over-Month Comparison -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monoMTitle') }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t('reports.table.month') }}</th>
                <th>{{ t('reports.table.orders') }}</th>
                <th>{{ t('reports.table.revenue') }}</th>
                <th>{{ t('reports.table.change') }}</th>
                <th>{{ t('reports.table.growthRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in monoMRows" :key="row.month">
                <td><strong>{{ row.label }}</strong></td>
                <td>{{ row.orders.toLocaleString(numberLocale) }}</td>
                <td>{{ formatCurrency(row.revenue, currentCurrency) }}</td>
                <td>
                  <span v-if="row.changeLabel !== null" :class="row.changeClass">{{ row.changeLabel }}</span>
                  <span v-else>-</span>
                </td>
                <td>
                  <span v-if="row.growthLabel !== null" :class="row.growthClass">{{ row.growthLabel }}</span>
                  <span v-else>-</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.totalRevenueYtd') }}</div>
          <div class="stat-value">{{ formatCurrency(totalRevenue, currentCurrency) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.avgMonthlyRevenue') }}</div>
          <div class="stat-value">{{ formatCurrency(avgMonthlyRevenue, currentCurrency) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.totalOrdersYtd') }}</div>
          <div class="stat-value">{{ totalOrders.toLocaleString(numberLocale) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.bestQuarter') }}</div>
          <div class="stat-value">{{ bestQuarter || t('reports.notAvailable') }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

// Month key order used to map YYYY-MM month numbers to locale keys
const MONTH_KEYS = ['jan', 'feb', 'mar', 'apr', 'may', 'jun', 'jul', 'aug', 'sep', 'oct', 'nov', 'dec']

export default {
  name: 'Reports',
  setup() {
    const { t, currentCurrency, currentLocale } = useI18n()

    const {
      selectedPeriod,
      selectedLocation,
      selectedCategory,
      selectedStatus,
      getCurrentFilters
    } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const quarterlyData = ref([])
    const monthlyData = ref([])

    // Derive number locale from app locale so toLocaleString renders correctly
    const numberLocale = computed(() => currentLocale.value === 'ja' ? 'ja-JP' : 'en-US')

    // Validates YYYY-MM shape and returns translated "Mon YYYY" label, or '-' on bad input
    const formatMonth = (monthStr) => {
      if (!monthStr || !/^\d{4}-\d{2}$/.test(monthStr)) return '-'
      const [year, monthNum] = monthStr.split('-')
      const idx = parseInt(monthNum, 10) - 1
      if (idx < 0 || idx > 11) return '-'
      return `${t(`months.${MONTH_KEYS[idx]}`)} ${year}`
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const [quarterly, monthly] = await Promise.all([
          api.getQuarterlyReports(filters),
          api.getMonthlyTrends(filters)
        ])
        quarterlyData.value = quarterly
        monthlyData.value = monthly
      } catch (err) {
        error.value = t('reports.loadError')
      } finally {
        loading.value = false
      }
    }

    // Watch all four global filter refs so the page stays in sync with the filter bar.
    // Backend only uses warehouse/category; period/status become no-ops on the server side.
    watch([selectedPeriod, selectedLocation, selectedCategory, selectedStatus], loadData)

    onMounted(loadData)

    // --- Computed derived state ---

    const maxMonthlyRevenue = computed(() =>
      monthlyData.value.reduce((max, m) => Math.max(max, m.revenue), 0)
    )

    const monthlyBars = computed(() =>
      monthlyData.value.map(m => ({
        month: m.month,
        revenue: m.revenue,
        height: maxMonthlyRevenue.value > 0 ? (m.revenue / maxMonthlyRevenue.value) * 200 : 0,
        label: formatCurrency(m.revenue, currentCurrency.value),
        monthLabel: formatMonth(m.month)
      }))
    )

    const monoMRows = computed(() =>
      monthlyData.value.map((m, index) => {
        if (index === 0) {
          return {
            month: m.month,
            label: formatMonth(m.month),
            orders: m.order_count,
            revenue: m.revenue,
            changeLabel: null,
            changeClass: '',
            growthLabel: null,
            growthClass: ''
          }
        }
        const prev = monthlyData.value[index - 1]
        const change = m.revenue - prev.revenue
        const changeLabel = change > 0
          ? '+' + formatCurrency(change, currentCurrency.value)
          : change < 0
            ? '-' + formatCurrency(Math.abs(change), currentCurrency.value)
            : formatCurrency(0, currentCurrency.value)
        const changeClass = change > 0 ? 'positive-change' : change < 0 ? 'negative-change' : ''

        let growthLabel
        let growthClass
        if (prev.revenue === 0) {
          growthLabel = t('reports.notAvailable')
          growthClass = ''
        } else {
          const rate = ((m.revenue - prev.revenue) / prev.revenue) * 100
          growthLabel = (rate > 0 ? '+' : '') + rate.toFixed(1) + '%'
          growthClass = rate > 0 ? 'positive-change' : rate < 0 ? 'negative-change' : ''
        }

        return {
          month: m.month,
          label: formatMonth(m.month),
          orders: m.order_count,
          revenue: m.revenue,
          changeLabel,
          changeClass,
          growthLabel,
          growthClass
        }
      })
    )

    const totalRevenue = computed(() =>
      monthlyData.value.reduce((sum, m) => sum + m.revenue, 0)
    )

    const avgMonthlyRevenue = computed(() =>
      monthlyData.value.length > 0 ? totalRevenue.value / monthlyData.value.length : 0
    )

    const totalOrders = computed(() =>
      monthlyData.value.reduce((sum, m) => sum + m.order_count, 0)
    )

    const bestQuarter = computed(() => {
      if (quarterlyData.value.length === 0) return ''
      return quarterlyData.value.reduce((best, q) =>
        q.total_revenue > best.total_revenue ? q : best
      ).quarter
    })

    const getFulfillmentClass = (rate) => {
      if (rate >= 90) return 'badge success'
      if (rate >= 75) return 'badge warning'
      return 'badge danger'
    }

    return {
      t,
      currentCurrency,
      loading,
      error,
      quarterlyData,
      monthlyData,
      numberLocale,
      monthlyBars,
      monoMRows,
      totalRevenue,
      avgMonthlyRevenue,
      totalOrders,
      bestQuarter,
      getFulfillmentClass,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.reports {
  padding: 0;
}

.card {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.card-header {
  margin-bottom: 1.5rem;
}

.card-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #0f172a;
  margin: 0;
}

.reports-table {
  width: 100%;
  border-collapse: collapse;
}

.reports-table th {
  background: #f8fafc;
  padding: 0.75rem;
  text-align: left;
  font-weight: 600;
  color: #64748b;
  border-bottom: 2px solid #e2e8f0;
}

.reports-table td {
  padding: 0.75rem;
  border-bottom: 1px solid #e2e8f0;
}

.reports-table tr:hover {
  background: #f8fafc;
}

.chart-container {
  padding: 2rem 1rem;
  min-height: 300px;
}

.bar-chart {
  display: flex;
  align-items: flex-end;
  justify-content: space-around;
  height: 250px;
  gap: 0.5rem;
}

.bar-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  max-width: 80px;
}

.bar-container {
  height: 200px;
  display: flex;
  align-items: flex-end;
  width: 100%;
}

.bar {
  width: 100%;
  background: linear-gradient(to top, #3b82f6, #60a5fa);
  border-radius: 4px 4px 0 0;
  transition: all 0.3s;
  cursor: pointer;
}

.bar:hover {
  background: linear-gradient(to top, #2563eb, #3b82f6);
}

.bar-label {
  margin-top: 1.5rem;
  font-size: 0.75rem;
  color: #64748b;
  text-align: center;
  transform: rotate(-45deg);
  white-space: nowrap;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
  margin-top: 1.5rem;
}

.stat-card {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border-left: 4px solid #3b82f6;
}

.stat-label {
  font-size: 0.875rem;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.stat-value {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
}

.badge {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
  font-weight: 500;
}

.badge.success {
  background: #dcfce7;
  color: #166534;
}

.badge.warning {
  background: #fef3c7;
  color: #92400e;
}

.badge.danger {
  background: #fee2e2;
  color: #991b1b;
}

.positive-change {
  color: #16a34a;
  font-weight: 600;
}

.negative-change {
  color: #dc2626;
  font-weight: 600;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.error {
  background: #fee2e2;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
}
</style>
