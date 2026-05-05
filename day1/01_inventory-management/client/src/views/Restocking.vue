<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and automatically generate a restocking order based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error && !successMessage" class="error">{{ error }}</div>
    <div v-else>
      <div
        v-if="successMessage"
        class="alert alert-success"
        role="alert"
      >
        {{ successMessage }}
      </div>

      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
          <span class="budget-display">{{ formatCurrency(budget) }}</span>
        </div>
        <div class="slider-wrapper">
          <label for="budget-slider" class="slider-label">Restocking Budget</label>
          <input
            id="budget-slider"
            type="range"
            min="0"
            max="500000"
            step="5000"
            v-model.number="budget"
            class="budget-slider"
            aria-label="Restocking budget"
            :aria-valuenow="budget"
            aria-valuemin="0"
            aria-valuemax="500000"
          />
          <div class="slider-bounds">
            <span>$0</span>
            <span>$500,000</span>
          </div>
        </div>
        <div class="budget-stats">
          <div class="stat-card info">
            <div class="stat-label">Total Cost</div>
            <div class="stat-value">{{ formatCurrency(totalCost) }}</div>
          </div>
          <div class="stat-card" :class="remainingBudget >= 0 ? 'success' : 'danger'">
            <div class="stat-label">Remaining Budget</div>
            <div class="stat-value">{{ formatCurrency(remainingBudget) }}</div>
          </div>
        </div>
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommended Items
            <span class="badge info count-badge">{{ recommendations.length }} items</span>
          </h3>
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item</th>
                <th>Trend</th>
                <th>Demand Gap</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Line Total</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="recommendations.length === 0">
                <td colspan="8" class="empty-state">
                  No items fit within this budget. Increase the budget to see recommendations.
                </td>
              </tr>
              <tr v-for="row in recommendations" :key="row.sku">
                <td><strong>{{ row.sku }}</strong></td>
                <td>{{ row.name }}</td>
                <td>
                  <span :class="['badge', trendBadgeClass(row.trend)]">{{ row.trend }}</span>
                </td>
                <td>{{ row.demand_gap }}</td>
                <td><strong>{{ row.quantity }}</strong></td>
                <td>{{ formatCurrency(row.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(row.line_total) }}</strong></td>
                <td>{{ row.lead_time_days }} days</td>
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
    const inventoryItems = ref([])
    const forecasts = ref([])
    const budget = ref(50000)
    const submitting = ref(false)
    const successMessage = ref('')
    const loading = ref(true)
    const error = ref(null)

    const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

    const recommendations = computed(() => {
      // Build sku -> inventory item lookup map
      const invMap = {}
      for (const inv of inventoryItems.value) {
        invMap[inv.sku] = inv
      }

      // Build candidate list
      const candidates = []
      for (const forecast of forecasts.value) {
        const inv = invMap[forecast.item_sku]
        if (!inv) continue

        const demand_gap = Math.max(0, forecast.forecasted_demand - inv.quantity_on_hand)
        if (demand_gap === 0) continue

        const line_total = demand_gap * inv.unit_cost

        candidates.push({
          sku: inv.sku,
          name: inv.name,
          trend: forecast.trend,
          demand_gap,
          quantity: demand_gap,
          unit_cost: inv.unit_cost,
          line_total,
          lead_time_days: inv.lead_time_days
        })
      }

      // Sort: trend priority (increasing → stable → decreasing), then line_total descending as tiebreaker
      candidates.sort((a, b) => {
        const trendDiff = (TREND_ORDER[a.trend] ?? 99) - (TREND_ORDER[b.trend] ?? 99)
        if (trendDiff !== 0) return trendDiff
        return b.line_total - a.line_total
      })

      // Greedy fill: walk sorted list, accumulate items within budget
      const result = []
      let remaining = budget.value

      for (const item of candidates) {
        if (remaining <= 0) break

        if (item.line_total <= remaining) {
          result.push({ ...item })
          remaining -= item.line_total
        } else {
          // Partially include the first item that doesn't fully fit
          const affordable_qty = Math.floor(remaining / item.unit_cost)
          if (affordable_qty > 0) {
            result.push({
              ...item,
              quantity: affordable_qty,
              line_total: affordable_qty * item.unit_cost
            })
          }
          break
        }
      }

      return result
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.line_total, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', {
        style: 'currency',
        currency: 'USD',
        maximumFractionDigits: 0
      })
    }

    const trendBadgeClass = (trend) => {
      if (trend === 'increasing') return 'success'
      if (trend === 'stable') return 'info'
      if (trend === 'decreasing') return 'warning'
      return ''
    }

    const placeOrder = async () => {
      submitting.value = true
      error.value = null
      try {
        const items = recommendations.value.map(r => ({
          item_sku: r.sku,
          item_name: r.name,
          quantity: r.quantity,
          unit_cost: r.unit_cost,
          lead_time_days: r.lead_time_days
        }))
        const order = await api.createRestockOrder(items)

        const deliveryDate = order.expected_delivery
          ? new Date(order.expected_delivery).toLocaleDateString('en-US', {
              year: 'numeric',
              month: 'long',
              day: 'numeric'
            })
          : 'TBD'

        successMessage.value = `Order ${order.order_number} submitted. Expected delivery: ${deliveryDate}.`
        budget.value = 0

        setTimeout(() => {
          successMessage.value = ''
        }, 3000)
      } catch (err) {
        error.value = 'Failed to submit restocking order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(async () => {
      loading.value = true
      error.value = null
      try {
        const [invData, forecastData] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventoryItems.value = invData
        forecasts.value = forecastData
      } catch (err) {
        error.value = 'Failed to load restocking data.'
        console.error(err)
      } finally {
        loading.value = false
      }
    })

    return {
      budget,
      submitting,
      successMessage,
      loading,
      error,
      recommendations,
      totalCost,
      remainingBudget,
      formatCurrency,
      trendBadgeClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  display: flex;
  align-items: center;
  gap: 0.625rem;
  margin: 0;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

/* Slider */
.slider-wrapper {
  padding: 0.5rem 0 1rem;
}

.slider-label {
  display: block;
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  -webkit-appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
  transition: background 0.2s;
  display: block;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: transform 0.15s, box-shadow 0.15s;
}

.budget-slider::-webkit-slider-thumb:hover {
  transform: scale(1.15);
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-slider:focus {
  outline: none;
}

.budget-slider:focus::-webkit-slider-thumb {
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.2), 0 1px 4px rgba(37, 99, 235, 0.4);
}

.slider-bounds {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Budget stats */
.budget-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1rem;
  margin-top: 0.5rem;
}

.stat-card {
  padding: 1rem 1.25rem;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  background: #f8fafc;
}

.stat-card.info {
  border-color: #bfdbfe;
  background: #eff6ff;
}

.stat-card.success {
  border-color: #a7f3d0;
  background: #f0fdf4;
}

.stat-card.danger {
  border-color: #fecaca;
  background: #fef2f2;
}

.stat-label {
  color: #64748b;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.375rem;
}

.stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.info .stat-value {
  color: #2563eb;
}

.stat-card.success .stat-value {
  color: #059669;
}

.stat-card.danger .stat-value {
  color: #dc2626;
}

/* Count badge in header */
.count-badge {
  font-size: 0.75rem;
  font-weight: 600;
}

/* Table */
.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

.empty-state {
  text-align: center;
  color: #94a3b8;
  font-style: italic;
  padding: 2.5rem 1rem;
}

/* Badges */
.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

/* Place Order button */
.btn-primary {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, opacity 0.2s;
  white-space: nowrap;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

/* Alert */
.alert {
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-bottom: 1.25rem;
}

.alert-success {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
}

/* Loading / Error */
.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
