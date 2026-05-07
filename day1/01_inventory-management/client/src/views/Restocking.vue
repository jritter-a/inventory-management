<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <template v-else>
      <div v-if="orderSuccess" class="success-banner">
        {{ t('restocking.orderPlaced') }}
      </div>

      <!-- Budget card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-control">
          <input
            type="range"
            :min="0"
            :max="totalPossibleCost"
            :step="Math.max(1, totalPossibleCost / 100)"
            v-model.number="budget"
          />
          <div class="budget-display">
            <span class="budget-amount">{{ formatCurrency(budget) }}</span>
            <span>{{ t('restocking.ofTotal') }} {{ formatCurrency(totalPossibleCost) }}</span>
          </div>
        </div>
      </div>

      <!-- Summary bar -->
      <div v-if="recommendations.length > 0" class="summary-bar card">
        <div class="summary-stat">
          <span class="summary-stat-label">{{ t('restocking.itemsSelected') }}</span>
          <span class="summary-stat-value">{{ summary.count }}</span>
        </div>
        <div class="summary-stat">
          <span class="summary-stat-label">{{ t('restocking.totalCost') }}</span>
          <span class="summary-stat-value">{{ formatCurrency(summary.totalCost) }}</span>
        </div>
        <div class="summary-stat">
          <span class="summary-stat-label">{{ t('restocking.remainingBudget') }}</span>
          <span class="summary-stat-value">{{ formatCurrency(summary.remainingBudget) }}</span>
        </div>
        <button
          class="place-order-btn"
          @click="placeOrder"
          :disabled="summary.count === 0 || placing"
        >
          {{ placing ? 'Placing...' : t('restocking.placeOrder') }}
        </button>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ recommendations.length }})</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.reason') }}</th>
                <th>{{ t('restocking.table.currentStock') }}</th>
                <th>{{ t('restocking.table.reorderPoint') }}</th>
                <th>{{ t('restocking.table.orderQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.totalCost') }}</th>
                <th>{{ t('restocking.table.status') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'over-budget': !selectedSkus.has(item.sku) }"
              >
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>
                  <span v-if="item.isBelowReorder" class="badge danger">
                    {{ t('restocking.reasons.belowReorder') }}
                  </span>
                  <span v-if="item.isIncreasing" class="badge success">
                    {{ t('restocking.reasons.increasingTrend') }}
                  </span>
                </td>
                <td>{{ item.quantityOnHand }}</td>
                <td>{{ item.reorderPoint }}</td>
                <td>{{ item.orderQty }}</td>
                <td>{{ formatCurrency(item.unitCost) }}</td>
                <td><strong>{{ formatCurrency(item.lineCost) }}</strong></td>
                <td>
                  <span v-if="selectedSkus.has(item.sku)" class="badge info">
                    {{ t('restocking.status.selected') }}
                  </span>
                  <span v-else class="badge" style="background:#f1f5f9;color:#94a3b8;">
                    {{ t('restocking.status.overBudget') }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <p v-if="recommendations.length === 0" class="no-data">
          {{ t('restocking.noRecommendations') }}
        </p>
      </div>
    </template>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api.js'
import { useI18n } from '../composables/useI18n.js'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const inventoryItems = ref([])
    const forecasts = ref([])
    const budget = ref(0)
    const placing = ref(false)
    const orderSuccess = ref(false)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [inventoryData, forecastData] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inventoryData
        forecasts.value = forecastData
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const recommendations = computed(() => {
      const forecastBySku = {}
      for (const f of forecasts.value) {
        forecastBySku[f.item_sku] = f
      }

      const result = []
      for (const item of inventoryItems.value) {
        const forecast = forecastBySku[item.sku]
        const isBelowReorder = item.quantity_on_hand < item.reorder_point
        const isIncreasing = forecast?.trend === 'increasing'

        if (!isBelowReorder && !isIncreasing) continue

        const orderQty = Math.max(
          0,
          (forecast?.forecasted_demand ?? 0) - (forecast?.current_demand ?? 0)
        )
        const lineCost = orderQty * item.unit_cost

        if (orderQty === 0 && lineCost === 0) continue

        result.push({
          sku: item.sku,
          name: item.name,
          category: item.category,
          warehouse: item.warehouse,
          quantityOnHand: item.quantity_on_hand,
          reorderPoint: item.reorder_point,
          unitCost: item.unit_cost,
          orderQty,
          lineCost,
          isBelowReorder,
          isIncreasing,
          urgency: isBelowReorder ? 0 : 1
        })
      }

      result.sort((a, b) => a.urgency - b.urgency)
      return result
    })

    const totalPossibleCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.lineCost, 0)
    )

    watch(totalPossibleCost, (val) => {
      if (budget.value === 0) budget.value = val
    }, { immediate: true })

    const selectedSkus = computed(() => {
      const selected = new Set()
      let remaining = budget.value
      for (const item of recommendations.value) {
        if (item.lineCost <= remaining) {
          selected.add(item.sku)
          remaining -= item.lineCost
        }
      }
      return selected
    })

    const summary = computed(() => {
      const selectedItems = recommendations.value.filter(i => selectedSkus.value.has(i.sku))
      const totalCost = selectedItems.reduce((sum, i) => sum + i.lineCost, 0)
      return {
        count: selectedItems.length,
        totalCost,
        remainingBudget: budget.value - totalCost
      }
    })

    const formatCurrency = (val) => {
      const symbol = currentCurrency.value === 'JPY' ? '¥' : '$'
      return `${symbol}${val.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
    }

    const placeOrder = async () => {
      placing.value = true
      orderSuccess.value = false
      try {
        const selected = recommendations.value.filter(i => selectedSkus.value.has(i.sku))
        await api.createOrder({
          customer: 'Internal Restock',
          items: selected.map(i => ({
            sku: i.sku,
            name: i.name,
            quantity: i.orderQty,
            unit_price: i.unitCost
          })),
          total_value: summary.value.totalCost,
          order_type: 'restocking'
        })
        orderSuccess.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      budget,
      placing,
      orderSuccess,
      recommendations,
      totalPossibleCost,
      selectedSkus,
      summary,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.over-budget {
  opacity: 0.45;
}

.budget-control {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: 0.5rem 0;
}

.budget-control input[type="range"] {
  width: 100%;
  accent-color: #3b82f6;
}

.budget-display {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
  color: #475569;
}

.budget-amount {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-bar {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.summary-stat {
  display: flex;
  flex-direction: column;
}

.summary-stat-label {
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
}

.summary-stat-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.place-order-btn {
  margin-left: auto;
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-weight: 500;
}

.no-data {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}
</style>
