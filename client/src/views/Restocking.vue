<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget.title') }}</h3>
        </div>
        <div class="budget-controls">
          <label class="budget-label" for="budget-slider">
            {{ t('restocking.budget.label') }}: <strong>{{ currencySymbol }}{{ budget.toLocaleString() }}</strong>
          </label>
          <input
            id="budget-slider"
            v-model.number="budget"
            type="range"
            min="0"
            max="100000"
            step="500"
            class="budget-slider"
          />
        </div>
      </div>

      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.budget.label') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.budget.selectedCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ selectedCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
        </div>
        <div class="stat-card" :class="remaining < 0 ? 'danger' : 'success'">
          <div class="stat-label">{{ t('restocking.budget.remaining') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ remaining.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('common.items') }}</div>
          <div class="stat-value">{{ selectedCount }}</div>
        </div>
      </div>

      <div v-if="successMessage" class="success-banner">{{ successMessage }}</div>
      <div v-if="orderError" class="error">{{ orderError }}</div>
      <div v-if="selectWarning" class="error">{{ selectWarning }}</div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendations.title') }}
            <span v-if="refreshing" class="refreshing-indicator">({{ t('common.loading') }})</span>
          </h3>
          <button
            class="place-order-btn"
            :disabled="submitting || selectedCount === 0"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placingOrder') : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="items.length === 0" class="no-data">
          {{ t('restocking.recommendations.noData') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th></th>
                <th>{{ t('restocking.recommendations.sku') }}</th>
                <th>{{ t('restocking.recommendations.itemName') }}</th>
                <th>{{ t('restocking.recommendations.category') }}</th>
                <th>{{ t('restocking.recommendations.warehouse') }}</th>
                <th>{{ t('restocking.recommendations.demand') }}</th>
                <th>{{ t('restocking.recommendations.quantityOnHand') }}</th>
                <th>{{ t('restocking.recommendations.reorderPoint') }}</th>
                <th>{{ t('restocking.recommendations.quantity') }}</th>
                <th>{{ t('restocking.recommendations.unitCost') }}</th>
                <th>{{ t('restocking.recommendations.itemCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in items" :key="item.sku">
                <td>
                  <input type="checkbox" v-model="item.checked" />
                </td>
                <td>
                  <strong>{{ item.sku }}</strong>
                  <span v-if="item.recommended" class="badge success recommended-badge">
                    {{ t('restocking.recommendations.recommended') }}
                  </span>
                </td>
                <td>{{ translateProductName(item.name) }}</td>
                <td>{{ translateCategory(item.category) }}</td>
                <td>{{ translateWarehouse(item.warehouse) }}</td>
                <td>{{ demandLabel(item) }}</td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>{{ item.reorder_point }}</td>
                <td>
                  <input
                    type="number"
                    min="1"
                    v-model.number="item.quantity"
                    class="quantity-input"
                    @blur="clampQuantity(item)"
                  />
                </td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td><strong>{{ currencySymbol }}{{ rowCost(item).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName, translateWarehouse } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(20000)
    const loading = ref(true)
    const refreshing = ref(false)
    const error = ref(null)
    const items = ref([])

    const submitting = ref(false)
    const orderError = ref(null)
    const selectWarning = ref(null)
    const successMessage = ref(null)

    let debounceTimer = null

    const selectedCost = computed(() => {
      return items.value
        .filter(item => item.checked)
        .reduce((sum, item) => sum + item.quantity * item.unit_cost, 0)
    })

    const remaining = computed(() => budget.value - selectedCost.value)

    const selectedCount = computed(() => items.value.filter(item => item.checked).length)

    const demandLabel = (item) => {
      if (item.current_demand == null && item.forecasted_demand == null) {
        return '—'
      }
      const current = item.current_demand ?? '—'
      const forecasted = item.forecasted_demand ?? '—'
      return `${current} / ${forecasted}`
    }

    const rowCost = (item) => item.quantity * item.unit_cost

    const clampQuantity = (item) => {
      if (!item.quantity || item.quantity < 1) {
        item.quantity = 1
      }
    }

    const translateCategory = (category) => {
      const categoryMap = {
        'Circuit Boards': t('categories.circuitBoards'),
        'Sensors': t('categories.sensors'),
        'Actuators': t('categories.actuators'),
        'Controllers': t('categories.controllers'),
        'Power Supplies': t('categories.powerSupplies')
      }
      return categoryMap[category] || category
    }

    const loadRecommendations = async () => {
      if (!loading.value) refreshing.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        items.value = data.items.map(item => ({
          ...item,
          checked: item.recommended,
          quantity: item.quantity
        }))
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
      } finally {
        loading.value = false
        refreshing.value = false
      }
    }

    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    const placeOrder = async () => {
      selectWarning.value = null
      orderError.value = null
      successMessage.value = null

      const selectedItems = items.value.filter(item => item.checked)
      if (selectedItems.length === 0) {
        selectWarning.value = t('restocking.selectAtLeastOne')
        return
      }

      submitting.value = true
      try {
        const order = await api.submitRestockingOrder({
          budget: budget.value,
          items: selectedItems.map(item => ({
            sku: item.sku,
            name: item.name,
            category: item.category,
            quantity: item.quantity,
            unit_cost: item.unit_cost
          }))
        })
        successMessage.value = t('restocking.orderSuccess', { id: order.id })
        await loadRecommendations()
      } catch (err) {
        orderError.value = t('restocking.orderError')
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)
    onUnmounted(() => clearTimeout(debounceTimer))

    return {
      t,
      budget,
      loading,
      refreshing,
      error,
      items,
      submitting,
      orderError,
      selectWarning,
      successMessage,
      selectedCost,
      remaining,
      selectedCount,
      demandLabel,
      rowCost,
      clampQuantity,
      translateCategory,
      translateProductName,
      translateWarehouse,
      placeOrder,
      currencySymbol
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-label {
  font-size: 0.938rem;
  color: #334155;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
}

.place-order-btn {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.refreshing-indicator {
  font-size: 0.813rem;
  font-weight: 400;
  color: #94a3b8;
  margin-left: 0.5rem;
}

.quantity-input {
  width: 70px;
  padding: 0.375rem 0.5rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.875rem;
}

.recommended-badge {
  margin-left: 0.5rem;
}

.no-data {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
