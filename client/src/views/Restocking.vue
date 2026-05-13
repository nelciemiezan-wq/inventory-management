<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card budget-card">
      <div class="budget-row">
        <label class="budget-label">{{ t('restocking.budgetLabel') }}</label>
        <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
      </div>
      <input
        type="range"
        min="0"
        max="500000"
        step="5000"
        v-model.number="budget"
        class="budget-slider"
      />
      <div class="budget-bounds">
        <span>{{ currencySymbol }}0</span>
        <span>{{ currencySymbol }}500,000</span>
      </div>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ recommendation.total_cost.toLocaleString() }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ recommendation.remaining_budget.toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.itemCount') }}</div>
          <div class="stat-value">{{ recommendation.items.length }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendation.items.length }})</h3>
        </div>
        <div v-if="recommendation.items.length === 0" class="empty-state">
          <p>{{ t('restocking.empty') }}</p>
        </div>
        <div v-else class="table-container">
          <table class="orders-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.name') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th>{{ t('restocking.table.recommendedQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
                <th>Trend</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendation.items" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ translateProductName(item.name) }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.quantity }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</strong></td>
                <td class="trend-cell">
                  <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="action-row">
          <div v-if="successMessage" class="success-banner">{{ successMessage }}</div>
          <button
            class="btn-primary"
            :disabled="recommendation.items.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()
    const router = useRouter()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const budget = ref(50000)
    const recommendation = ref({ budget: 0, items: [], total_cost: 0, remaining_budget: 0 })
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')

    let debounceTimer = null
    const fetchRecommendations = () => {
      // Why: <input type=range> fires 'input' every pixel; debounce so a quick
      // drag doesn't spam the API and cause out-of-order responses.
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(async () => {
        try {
          loading.value = true
          recommendation.value = await api.getRestockingRecommendations(budget.value)
          error.value = null
        } catch (e) {
          error.value = 'Failed to load recommendations: ' + e.message
        } finally {
          loading.value = false
        }
      }, 250)
    }

    watch(budget, fetchRecommendations, { immediate: false })
    onMounted(fetchRecommendations)

    const placeOrder = async () => {
      if (!recommendation.value.items.length) return
      submitting.value = true
      try {
        const items = recommendation.value.items.map(r => ({
          sku: r.sku,
          name: r.name,
          quantity: r.quantity,
          unit_price: r.unit_cost
        }))
        await api.createOrder({ items })
        successMessage.value = t('restocking.success')
        setTimeout(() => router.push('/orders'), 800)
      } catch (e) {
        error.value = 'Failed to place order: ' + e.message
      } finally {
        submitting.value = false
      }
    }

    return {
      t,
      currencySymbol,
      translateProductName,
      budget,
      recommendation,
      loading,
      error,
      submitting,
      successMessage,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
  padding: 1.5rem;
}

.budget-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 500;
  color: #64748b;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  height: 6px;
  margin-bottom: 0.5rem;
}

.budget-bounds {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 0.25rem;
}

.action-row {
  margin-top: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.75rem;
  padding: 0 1rem 1rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  width: 100%;
  background: #dcfce7;
  color: #166534;
  border: 1px solid #22c55e;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  font-weight: 500;
  text-align: center;
}

.empty-state {
  text-align: center;
  color: #64748b;
  padding: 2rem;
  font-size: 0.9rem;
}

.trend-cell {
  white-space: nowrap;
}
</style>
