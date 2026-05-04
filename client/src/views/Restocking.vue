<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()

    const budget = ref(25000)
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const selectedSkus = ref(new Set())
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const orderSuccess = ref(null)

    // Build a sku -> unit_cost map from inventory data
    const costMap = computed(() => {
      const map = new Map()
      inventoryItems.value.forEach(item => {
        map.set(item.sku, item.unit_cost)
      })
      return map
    })

    // Greedy budget allocation: sort by forecasted_demand desc, fill budget
    const recommendedItems = computed(() => {
      const sorted = [...demandForecasts.value].sort(
        (a, b) => b.forecasted_demand - a.forecasted_demand
      )

      const results = []
      let remaining = budget.value

      for (const item of sorted) {
        if (remaining <= 0) break
        const unit_cost = costMap.value.get(item.item_sku) ?? 10.0
        const max_qty = Math.min(
          item.forecasted_demand,
          Math.floor(remaining / unit_cost)
        )
        if (max_qty > 0) {
          results.push({
            ...item,
            unit_cost,
            recommended_qty: max_qty,
            total_cost: max_qty * unit_cost
          })
          remaining -= max_qty * unit_cost
        }
      }

      return results
    })

    // Budget remaining after all recommended items
    const budgetRemaining = computed(() => {
      const used = recommendedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
      return budget.value - used
    })

    // Cost of only the selected items
    const selectedTotalCost = computed(() => {
      return recommendedItems.value
        .filter(item => selectedSkus.value.has(item.item_sku))
        .reduce((sum, item) => sum + item.total_cost, 0)
    })

    const selectedBudgetUsedPct = computed(() => {
      if (budget.value === 0) return 0
      return Math.round((selectedTotalCost.value / budget.value) * 100)
    })

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const toggleSku = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) {
        next.delete(sku)
      } else {
        next.add(sku)
      }
      selectedSkus.value = next
    }

    const isSkuSelected = (sku) => selectedSkus.value.has(sku)

    const placeOrder = async () => {
      const itemsToOrder = recommendedItems.value
        .filter(item => selectedSkus.value.has(item.item_sku))
        .map(item => ({
          item_sku: item.item_sku,
          item_name: item.item_name,
          quantity: item.recommended_qty,
          unit_cost: item.unit_cost,
          total_cost: item.total_cost
        }))
      const total_cost = itemsToOrder.reduce((sum, i) => sum + i.total_cost, 0)
      submitting.value = true
      try {
        orderSuccess.value = await api.createRestockingOrder({
          items: itemsToOrder,
          total_cost: Math.round(total_cost * 100) / 100,
          budget: budget.value
        })
        // Reset selection after successful order
        selectedSkus.value = new Set()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    // When recommendedItems change, default-select all recommended SKUs
    watch(
      recommendedItems,
      (items) => {
        selectedSkus.value = new Set(items.map(item => item.item_sku))
      },
      { deep: true }
    )

    onMounted(() => loadData())

    return {
      t,
      budget,
      demandForecasts,
      inventoryItems,
      selectedSkus,
      loading,
      error,
      submitting,
      orderSuccess,
      recommendedItems,
      budgetRemaining,
      selectedTotalCost,
      selectedBudgetUsedPct,
      toggleSku,
      isSkuSelected,
      placeOrder,
      formatDate
    }
  }
}
</script>

<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Allocate your budget to restock high-demand items</p>
    </div>

    <!-- Success banner -->
    <div v-if="orderSuccess" class="success-banner">
      <div class="success-banner-content">
        <strong>Order {{ orderSuccess.order_number }} placed successfully!</strong>
        Estimated delivery: {{ formatDate(orderSuccess.estimated_delivery) }}
      </div>
      <router-link to="/orders" class="success-link">View in Orders tab</router-link>
    </div>

    <!-- Budget card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
      </div>
      <div class="budget-body">
        <div class="budget-value">${{ budget.toLocaleString() }}</div>
        <input
          type="range"
          v-model.number="budget"
          min="0"
          max="50000"
          step="500"
          class="budget-slider"
        />
        <div class="budget-remaining">
          Budget remaining after selections: <strong>${{ budgetRemaining.toLocaleString() }}</strong>
        </div>
      </div>
    </div>

    <!-- Recommendations card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">
          Recommended Items ({{ recommendedItems.length }})
        </h3>
        <span class="selected-cost-badge">
          Selected: ${{ selectedTotalCost.toLocaleString() }}
        </span>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendedItems.length === 0" class="empty-state">
        Increase your budget to see recommendations.
      </div>
      <div v-else>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-name">Item Name</th>
                <th class="col-sku">SKU</th>
                <th class="col-trend">Trend</th>
                <th class="col-number">Forecasted Demand</th>
                <th class="col-number">Recommended Qty</th>
                <th class="col-currency">Unit Cost</th>
                <th class="col-currency">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="isSkuSelected(item.item_sku)"
                    @change="toggleSku(item.item_sku)"
                    class="row-checkbox"
                  />
                </td>
                <td class="col-name">{{ item.item_name }}</td>
                <td class="col-sku">{{ item.item_sku }}</td>
                <td class="col-trend">
                  <span :class="['badge', item.trend.toLowerCase()]">
                    {{ item.trend }}
                  </span>
                </td>
                <td class="col-number">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-number">{{ item.recommended_qty.toLocaleString() }}</td>
                <td class="col-currency">${{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-currency"><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Order summary bar -->
        <div class="order-summary-bar">
          <div class="order-summary-info">
            <span>{{ selectedSkus.size }} item(s) selected</span>
            <span class="summary-divider">|</span>
            <span>Total: ${{ selectedTotalCost.toLocaleString() }}</span>
            <span class="summary-divider">|</span>
            <span>Budget used: {{ selectedBudgetUsedPct }}%</span>
          </div>
          <button
            class="btn-place-order"
            :disabled="selectedSkus.size === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.restocking {
  padding: 0;
}

/* Success banner */
.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.success-banner-content {
  flex: 1;
}

.success-link {
  margin-left: 1.5rem;
  color: #059669;
  font-weight: 600;
  text-decoration: underline;
  white-space: nowrap;
}

.success-link:hover {
  color: #047857;
}

/* Budget card body */
.budget-body {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding: 0.25rem 0;
}

.budget-value {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  max-width: 600px;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-remaining {
  font-size: 0.875rem;
  color: #64748b;
}

/* Selected cost badge in card header */
.selected-cost-badge {
  font-size: 0.875rem;
  font-weight: 600;
  color: #1e40af;
  background: #dbeafe;
  padding: 0.313rem 0.875rem;
  border-radius: 6px;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Recommendations table */
.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-check {
  width: 40px;
}

.col-name {
  width: 220px;
}

.col-sku {
  width: 120px;
}

.col-trend {
  width: 110px;
}

.col-number {
  width: 150px;
}

.col-currency {
  width: 110px;
}

.row-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

/* Order summary bar */
.order-summary-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.875rem 0.75rem;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
  border-radius: 0 0 8px 8px;
  margin-top: 0.5rem;
}

.order-summary-info {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.875rem;
  color: #334155;
  font-weight: 500;
}

.summary-divider {
  color: #cbd5e1;
  font-weight: 400;
}

.btn-place-order {
  background: #2563eb;
  color: #ffffff;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
