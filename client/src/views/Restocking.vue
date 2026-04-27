<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Demand-weighted purchasing plan based on inventory levels and demand forecasts</p>
    </div>

    <div class="budget-row">
      <label class="budget-label" for="budget-input">Budget</label>
      <input
        id="budget-input"
        v-model.number="budget"
        type="number"
        min="1"
        step="100"
        class="budget-input"
      />
      <button
        class="recommend-btn"
        :disabled="loading"
        @click="loadRestocking"
      >
        Get Recommendations
      </button>
      <span class="budget-hint">Enter a total budget to generate a prioritized restocking plan</span>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <template v-else-if="hasResults">
      <div v-if="result.recommendations.length === 0" class="card empty-card">
        <p class="empty-text">No items require restocking within this budget.</p>
      </div>
      <template v-else>
        <div class="stats-grid">
          <div class="stat-card info">
            <div class="stat-label">Total Est. Cost</div>
            <div class="stat-value">{{ currencySymbol }}{{ result.total_estimated_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
          </div>
          <div :class="['stat-card', result.budget_remaining >= 0 ? 'success' : 'danger']">
            <div class="stat-label">Budget Remaining</div>
            <div class="stat-value">{{ currencySymbol }}{{ result.budget_remaining.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
          </div>
          <div class="stat-card">
            <div class="stat-label">Items Addressed</div>
            <div class="stat-value">{{ result.items_addressed }}</div>
          </div>
        </div>

        <div class="card">
          <div class="card-header">
            <h3 class="card-title">Recommendations ({{ result.recommendations.length }} items)</h3>
          </div>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>SKU</th>
                  <th>Item Name</th>
                  <th>Category</th>
                  <th>Warehouse</th>
                  <th>On Hand</th>
                  <th>Reorder Pt</th>
                  <th>Trend</th>
                  <th>Rec. Qty</th>
                  <th>Est. Cost</th>
                  <th>Priority</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in result.recommendations" :key="item.id">
                  <td><strong>{{ item.sku }}</strong></td>
                  <td>{{ item.name }}</td>
                  <td>{{ item.category }}</td>
                  <td>{{ item.warehouse }}</td>
                  <td>
                    <strong :class="{ 'on-hand-low': item.quantity_on_hand <= item.reorder_point }">
                      {{ item.quantity_on_hand }}
                    </strong>
                  </td>
                  <td>{{ item.reorder_point }}</td>
                  <td>
                    <span v-if="item.trend" :class="['badge', item.trend]">{{ item.trend }}</span>
                  </td>
                  <td><strong>{{ item.recommended_qty }}</strong></td>
                  <td><strong>{{ currencySymbol }}{{ item.estimated_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
                  <td><span :class="['badge', item.priority]">{{ item.priority }}</span></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </template>
    </template>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const budget = ref(5000)
    const loading = ref(false)
    const error = ref(null)
    const result = ref(null)
    const hasResults = ref(false)

    const loadRestocking = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        result.value = await api.getRestocking(budget.value, {
          warehouse: filters.warehouse,
          category: filters.category
        })
        hasResults.value = true
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch([selectedLocation, selectedCategory], () => {
      if (hasResults.value) {
        loadRestocking()
      }
    })

    return {
      currencySymbol,
      budget,
      loading,
      error,
      result,
      hasResults,
      loadRestocking
    }
  }
}
</script>

<style scoped>
.budget-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #0f172a;
  white-space: nowrap;
}

.budget-input {
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 0.5rem 0.75rem;
  font-size: 0.875rem;
  color: #0f172a;
  width: 140px;
}

.budget-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.recommend-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
  white-space: nowrap;
}

.recommend-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.recommend-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.budget-hint {
  font-size: 0.813rem;
  color: #64748b;
}

.empty-card {
  text-align: center;
  padding: 2.5rem 1.25rem;
}

.empty-text {
  color: #64748b;
  font-size: 0.938rem;
}

.on-hand-low {
  color: #dc2626;
}
</style>
