<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card budget-card">
      <div class="budget-control">
        <label for="budget-slider" class="budget-label">{{ t('restocking.budgetLabel') }}</label>
        <input
          id="budget-slider"
          type="range"
          min="0"
          max="50000"
          step="500"
          v-model.number="budget"
          class="budget-slider"
        />
        <span class="budget-value">{{ formatCurrency(budget, currentCurrency) }}</span>
      </div>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="submitSuccess" class="success-banner">
        {{ t('restocking.successMessage', { orderNumber: submitSuccess.order_number }) }}
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        </div>

        <div v-if="!recommendation || !recommendation.line_items.length" class="empty-state">
          {{ t('restocking.noItems') }}
        </div>
        <template v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>{{ t('restocking.table.sku') }}</th>
                  <th>{{ t('restocking.table.itemName') }}</th>
                  <th>{{ t('restocking.table.trend') }}</th>
                  <th>{{ t('restocking.table.shortfall') }}</th>
                  <th>{{ t('restocking.table.unitCost') }}</th>
                  <th>{{ t('restocking.table.quantity') }}</th>
                  <th>{{ t('restocking.table.lineTotal') }}</th>
                  <th>{{ t('restocking.table.leadTime') }}</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="li in recommendation.line_items" :key="li.item_sku">
                  <td><strong>{{ li.item_sku }}</strong></td>
                  <td>
                    {{ li.item_name }}
                    <span v-if="li.is_partial" class="badge partial-badge">{{ t('restocking.partialBadge') }}</span>
                  </td>
                  <td>
                    <span :class="['badge', li.trend]">{{ t(`trends.${li.trend}`) }}</span>
                  </td>
                  <td>{{ li.shortfall }}</td>
                  <td>{{ formatCurrency(li.unit_cost, currentCurrency) }}</td>
                  <td>{{ li.quantity }}</td>
                  <td><strong>{{ formatCurrency(li.line_total, currentCurrency) }}</strong></td>
                  <td>{{ li.lead_time_days }} days</td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="summary-line">
            <span>{{ t('restocking.totalCost') }}: <strong>{{ formatCurrency(recommendation.total_cost, currentCurrency) }}</strong></span>
            <span>{{ t('restocking.remainingBudget') }}: <strong>{{ formatCurrency(recommendation.remaining_budget, currentCurrency) }}</strong></span>
          </div>
        </template>

        <button
          class="place-order-btn"
          :disabled="submitting || !recommendation || !recommendation.line_items.length"
          @click="submitOrder"
        >
          {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const budget = ref(10000)
    const recommendation = ref(null)
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const submitSuccess = ref(null)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendation.value = await api.getRestockRecommendations(budget.value)
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const submitOrder = async () => {
      try {
        submitting.value = true
        submitSuccess.value = await api.createRestockOrder(budget.value)
      } catch (err) {
        error.value = 'Failed to submit restocking order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    watch(budget, loadRecommendations)
    onMounted(loadRecommendations)

    return {
      t,
      currentCurrency,
      budget,
      recommendation,
      loading,
      error,
      submitting,
      submitSuccess,
      formatCurrency,
      submitOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-control {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-label {
  font-weight: 600;
  color: #475569;
  font-size: 0.875rem;
  flex-shrink: 0;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
}

.budget-value {
  font-weight: 700;
  color: #0f172a;
  font-size: 1.125rem;
  min-width: 90px;
  text-align: right;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.partial-badge {
  margin-left: 0.5rem;
  background: #fed7aa;
  color: #92400e;
}

.summary-line {
  display: flex;
  justify-content: flex-end;
  gap: 2rem;
  padding: 1rem 0.75rem;
  font-size: 0.938rem;
  color: #334155;
}

.place-order-btn {
  display: block;
  margin: 1.25rem 0 0 auto;
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>
