<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Allocate a budget and get recommendations based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-controls">
          <input
            type="range"
            v-model.number="budget"
            min="0"
            max="500000"
            step="5000"
            class="budget-slider"
          />
          <span class="budget-display">${{ budget.toLocaleString() }}</span>
        </div>
      </div>

      <!-- Success / Error Banner -->
      <div v-if="orderSuccess" class="banner success-banner">
        Order <strong>{{ orderSuccess.order_number }}</strong> placed successfully.
        Expected delivery in {{ orderSuccess.delivery_lead_time_days }} days.
      </div>
      <div v-if="orderError" class="banner error-banner">{{ orderError }}</div>

      <!-- Recommended Items Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ inBudgetCount }} selected, total ${{ runningTotal.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }})</h3>
          <button
            class="btn-primary"
            :disabled="inBudgetCount === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing...' : 'Place Order' }}
          </button>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
                <th>In Budget?</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="recommendations.length === 0">
                <td colspan="6" style="text-align:center; color:#64748b;">No demand growth items found.</td>
              </tr>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'row-in-budget': item.in_budget, 'row-out': !item.in_budget }"
              >
                <td><code>{{ item.sku }}</code></td>
                <td>{{ item.name }}</td>
                <td>{{ item.restock_qty }}</td>
                <td>
                  <span v-if="item.unit_cost === 0" class="text-muted">N/A</span>
                  <span v-else>${{ item.unit_cost.toFixed(2) }}</span>
                </td>
                <td>
                  <span v-if="item.unit_cost === 0" class="text-muted">—</span>
                  <span v-else>${{ item.line_total.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
                </td>
                <td>
                  <span v-if="item.unit_cost === 0" class="badge info">No Cost Data</span>
                  <span v-else-if="item.in_budget" class="badge success">Yes</span>
                  <span v-else class="badge danger">No</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const budget = ref(50000)
    const forecasts = ref([])
    const unitCostMap = ref({})
    const placing = ref(false)
    const orderSuccess = ref(null)
    const orderError = ref(null)

    onMounted(async () => {
      try {
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastData

        // Build SKU -> unit_cost lookup from inventory
        const costMap = {}
        for (const inv of inventoryData) {
          costMap[inv.sku] = inv.unit_cost
        }
        unitCostMap.value = costMap
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    })

    // Compute recommendations sorted by demand growth %, greedy budget allocation
    const recommendations = computed(() => {
      const items = forecasts.value
        .filter(f => f.forecasted_demand > f.current_demand && f.current_demand > 0)
        .map(f => {
          const unit_cost = unitCostMap.value[f.item_sku] ?? 0
          const demand_growth = f.forecasted_demand - f.current_demand
          const restock_qty = Math.max(1, demand_growth)
          const line_total = restock_qty * unit_cost
          const growth_pct = demand_growth / f.current_demand
          return {
            sku: f.item_sku,
            name: f.item_name,
            restock_qty,
            unit_cost,
            line_total,
            growth_pct,
            in_budget: false
          }
        })
        .sort((a, b) => b.growth_pct - a.growth_pct)

      // Greedy budget fill — items with no cost data are skipped
      let running = 0
      for (const item of items) {
        if (item.unit_cost === 0) {
          item.in_budget = false
          continue
        }
        if (running + item.line_total <= budget.value) {
          item.in_budget = true
          running += item.line_total
        } else {
          item.in_budget = false
        }
      }

      return items
    })

    const inBudgetCount = computed(() => recommendations.value.filter(r => r.in_budget).length)

    const runningTotal = computed(() =>
      recommendations.value
        .filter(r => r.in_budget)
        .reduce((sum, r) => sum + r.line_total, 0)
    )

    const placeOrder = async () => {
      orderSuccess.value = null
      orderError.value = null
      placing.value = true
      try {
        const items = recommendations.value
          .filter(r => r.in_budget)
          .map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.restock_qty,
            unit_cost: r.unit_cost
          }))
        const result = await api.createRestockingOrder({ items })
        orderSuccess.value = result
      } catch (err) {
        orderError.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    return {
      loading,
      error,
      budget,
      recommendations,
      inBudgetCount,
      runningTotal,
      placing,
      orderSuccess,
      orderError,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-controls {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 0.5rem 0;
}

.budget-slider {
  flex: 1;
  max-width: 480px;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 140px;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.banner {
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.875rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
}

.error-banner {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
}

.row-in-budget {
  background: #f0fdf4;
}

.row-out {
  opacity: 0.65;
}

.text-muted {
  color: #94a3b8;
}

code {
  font-family: monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
}
</style>
