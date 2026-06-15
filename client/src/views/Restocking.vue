<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and get item recommendations based on demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading restocking data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="submittedOrder">
      <div class="card success-card">
        <div class="success-checkmark">&#10003;</div>
        <h3 class="success-title">Order Submitted Successfully</h3>
        <p class="success-order-number">Order #{{ submittedOrder.order_id || submittedOrder.id }}</p>
        <p class="success-delivery">Expected delivery in 30 days</p>
        <div class="success-actions">
          <router-link to="/orders" class="btn-primary">View in Orders tab</router-link>
          <button class="btn-secondary" @click="resetForm">Place Another Order</button>
        </div>
      </div>
    </div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
          <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <div class="slider-wrapper">
          <input
            type="range"
            min="0"
            max="500000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="slider-labels">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}500,000</span>
          </div>
        </div>
        <p class="remaining-after">
          Remaining after recommendations:
          <strong :style="{ color: remainingBudget < 0 ? '#dc2626' : '#059669' }">
            {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}
          </strong>
        </p>
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
        </div>

        <div v-if="recommendations.length === 0" style="padding: 2rem; text-align: center; color: #64748b;">
          All items are meeting their forecasted demand.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Current</th>
                <th>Forecasted</th>
                <th>Gap</th>
                <th>Unit Cost</th>
                <th>Restock Qty</th>
                <th>Est. Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-recommended': item.recommended, 'row-over-budget': !item.recommended }"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.current_demand }}</td>
                <td><strong>{{ item.forecasted_demand }}</strong></td>
                <td>{{ item.gap }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td>{{ item.restock_qty }}</td>
                <td>{{ currencySymbol }}{{ item.estimated_cost.toLocaleString() }}</td>
                <td>
                  <span v-if="item.recommended" class="badge success">Recommended</span>
                  <span v-else class="badge danger">Over budget</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary Bar -->
        <div class="summary-bar">
          <div class="summary-stats">
            <span class="summary-stat">
              <strong>{{ recommendedItems.length }}</strong> items recommended
            </span>
            <span class="summary-divider">|</span>
            <span class="summary-stat">
              Total cost: <strong>{{ currencySymbol }}{{ totalCost.toLocaleString() }}</strong>
            </span>
            <span class="summary-divider">|</span>
            <span class="summary-stat">
              Remaining budget:
              <strong :style="{ color: remainingBudget < 0 ? '#dc2626' : '#059669' }">
                {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}
              </strong>
            </span>
          </div>
          <button
            class="btn-place-order"
            :disabled="submitting || recommendedItems.length === 0"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const budget = ref(100000)
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submittedOrder = ref(null)
    const allForecasts = ref([])
    const inventoryItems = ref([])

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const enrichedForecasts = computed(() => {
      const inventoryMap = {}
      inventoryItems.value.forEach(item => { inventoryMap[item.sku] = item })

      return allForecasts.value
        .map(forecast => {
          const invItem = inventoryMap[forecast.item_sku]
          const unit_cost = invItem ? invItem.unit_cost : null
          const gap = forecast.forecasted_demand - forecast.current_demand
          const restock_qty = gap
          const estimated_cost = unit_cost !== null ? unit_cost * restock_qty : null
          return { ...forecast, unit_cost, gap, restock_qty, estimated_cost }
        })
        .filter(f => f.gap > 0 && f.unit_cost !== null)
        .sort((a, b) => b.gap - a.gap)
    })

    const recommendations = computed(() => {
      let remaining = budget.value
      return enrichedForecasts.value.map(item => {
        const fits = item.estimated_cost <= remaining
        if (fits) remaining -= item.estimated_cost
        return { ...item, recommended: fits }
      })
    })

    const recommendedItems = computed(() => recommendations.value.filter(i => i.recommended))
    const totalCost = computed(() => recommendedItems.value.reduce((sum, i) => sum + i.estimated_cost, 0))
    const remainingBudget = computed(() => budget.value - totalCost.value)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0) return
      try {
        submitting.value = true
        error.value = null
        const items = recommendedItems.value.map(i => ({
          sku: i.item_sku,
          name: i.item_name,
          quantity: i.restock_qty,
          unit_price: i.unit_cost
        }))
        const result = await api.createRestockOrder({
          items,
          total_value: totalCost.value
        })
        submittedOrder.value = result
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const resetForm = () => {
      submittedOrder.value = null
      budget.value = 100000
    }

    onMounted(loadData)

    return {
      budget,
      loading,
      error,
      submitting,
      submittedOrder,
      recommendations,
      recommendedItems,
      totalCost,
      remainingBudget,
      currencySymbol,
      placeOrder,
      resetForm
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-wrapper {
  padding: 0.5rem 0 0.25rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.75rem;
  color: #64748b;
}

.remaining-after {
  margin-top: 0.75rem;
  font-size: 0.875rem;
  color: #64748b;
}

.row-recommended {
  background: #f0fdf4;
  border-left: 3px solid #059669;
}

.row-over-budget {
  opacity: 0.45;
}

.summary-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem;
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-radius: 0 0 10px 10px;
  margin: 0 -1.25rem -1.25rem;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.summary-stats {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.summary-stat {
  font-size: 0.875rem;
  color: #334155;
}

.summary-divider {
  color: #e2e8f0;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  border: none;
  cursor: pointer;
  font-size: 0.875rem;
  transition: background 0.2s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-card {
  text-align: center;
  padding: 3rem 2rem;
  border-left: 4px solid #059669;
}

.success-checkmark {
  width: 64px;
  height: 64px;
  background: #d1fae5;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1.25rem;
  font-size: 2rem;
  color: #059669;
  font-weight: 700;
  line-height: 64px;
}

.success-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.5rem;
}

.success-order-number {
  font-size: 1rem;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.success-delivery {
  font-size: 0.938rem;
  color: #64748b;
  margin-bottom: 1.75rem;
}

.success-actions {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.btn-primary {
  background: #2563eb;
  color: white;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.875rem;
  text-decoration: none;
  transition: background 0.2s ease;
  display: inline-block;
}

.btn-primary:hover {
  background: #1d4ed8;
}

.btn-secondary {
  background: white;
  color: #334155;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.875rem;
  border: 1px solid #e2e8f0;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-secondary:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}
</style>
