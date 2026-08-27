<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <div class="item-header">
              <div class="item-title-section">
                <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
            </div>

            <!-- CREATE MODE -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitForm">
              <div class="form-group">
                <label for="supplier_name">Supplier Name</label>
                <input
                  id="supplier_name"
                  v-model="form.supplier_name"
                  type="text"
                  required
                  placeholder="e.g. Acme Components Inc."
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label for="quantity">Quantity</label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label for="unit_cost">Unit Cost</label>
                  <input
                    id="unit_cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    min="0"
                    step="0.01"
                    required
                  />
                </div>
              </div>

              <div class="form-group">
                <label for="expected_delivery_date">Expected Delivery Date</label>
                <input
                  id="expected_delivery_date"
                  v-model="form.expected_delivery_date"
                  type="date"
                  required
                />
              </div>

              <div class="form-group">
                <label for="notes">Notes (optional)</label>
                <textarea
                  id="notes"
                  v-model="form.notes"
                  rows="3"
                  placeholder="Additional notes for this order..."
                ></textarea>
              </div>

              <div v-if="submitError" class="form-error">{{ submitError }}</div>
            </form>

            <!-- VIEW MODE -->
            <div v-else class="po-view">
              <div v-if="viewLoading" class="view-status">Loading purchase order...</div>
              <div v-else-if="viewError" class="view-status error">{{ viewError }}</div>
              <div v-else-if="purchaseOrder" class="info-grid">
                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ purchaseOrder.supplier_name }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="badge" :class="statusBadgeClass(purchaseOrder.status)">
                      {{ purchaseOrder.status }}
                    </span>
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ purchaseOrder.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ Number(purchaseOrder.unit_cost).toFixed(2) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value">${{ totalCost.toFixed(2) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(purchaseOrder.expected_delivery_date) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Created</div>
                  <div class="info-value">{{ formatDate(purchaseOrder.created_date) }}</div>
                </div>

                <div v-if="purchaseOrder.notes" class="info-item info-item-full">
                  <div class="info-label">Notes</div>
                  <div class="info-value">{{ purchaseOrder.notes }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <template v-if="mode === 'create'">
              <button type="button" class="btn-secondary" @click="close">Cancel</button>
              <button type="button" class="btn-primary" :disabled="submitting" @click="submitForm">
                {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
              </button>
            </template>
            <template v-else>
              <button type="button" class="btn-secondary" @click="close">Close</button>
            </template>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create'
  }
})

const emit = defineEmits(['close', 'po-created'])

const defaultForm = () => ({
  supplier_name: '',
  quantity: null,
  unit_cost: null,
  expected_delivery_date: '',
  notes: ''
})

const form = ref(defaultForm())
const submitting = ref(false)
const submitError = ref(null)

const purchaseOrder = ref(null)
const viewLoading = ref(false)
const viewError = ref(null)

const totalCost = computed(() => {
  if (!purchaseOrder.value) return 0
  return Number(purchaseOrder.value.quantity) * Number(purchaseOrder.value.unit_cost)
})

const resetState = () => {
  form.value = defaultForm()
  submitting.value = false
  submitError.value = null
  purchaseOrder.value = null
  viewLoading.value = false
  viewError.value = null
}

const prefillForm = () => {
  if (!props.backlogItem) return
  const shortage = props.backlogItem.quantity_needed - props.backlogItem.quantity_available
  form.value = {
    ...defaultForm(),
    quantity: shortage > 0 ? shortage : 1
  }
}

const fetchExistingPO = async () => {
  if (!props.backlogItem) return
  viewLoading.value = true
  viewError.value = null
  purchaseOrder.value = null
  try {
    purchaseOrder.value = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
  } catch (err) {
    if (err.response && err.response.status === 404) {
      viewError.value = 'No purchase order found for this item.'
    } else {
      viewError.value = 'Failed to load purchase order.'
    }
    console.error(err)
  } finally {
    viewLoading.value = false
  }
}

const syncModalState = () => {
  if (!props.isOpen) {
    resetState()
    return
  }
  if (props.mode === 'create') {
    resetState()
    prefillForm()
  } else {
    resetState()
    fetchExistingPO()
  }
}

// Watch isOpen/mode/backlogItem together so re-opening or switching mode/item
// while the modal stays mounted always refreshes state (never shows stale data)
watch(
  () => [props.isOpen, props.mode, props.backlogItem],
  () => {
    syncModalState()
  },
  { immediate: true }
)

const submitForm = async () => {
  if (!props.backlogItem) return
  submitting.value = true
  submitError.value = null
  try {
    const payload = {
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplier_name,
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery_date: form.value.expected_delivery_date
    }
    if (form.value.notes) {
      payload.notes = form.value.notes
    }
    const created = await api.createPurchaseOrder(payload)
    emit('po-created', created)
    emit('close')
  } catch (err) {
    if (err.response && err.response.data && err.response.data.detail) {
      submitError.value = err.response.data.detail
    } else {
      submitError.value = 'Failed to create purchase order.'
    }
    console.error(err)
  } finally {
    submitting.value = false
  }
}

const close = () => {
  emit('close')
}

const statusBadgeClass = (status) => {
  const s = (status || '').toLowerCase()
  if (s === 'delivered' || s === 'received' || s === 'completed') return 'success'
  if (s === 'shipped') return 'info'
  if (s === 'pending' || s === 'ordered' || s === 'processing') return 'warning'
  return 'danger'
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return 'N/A'
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

.item-header {
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.item-name {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.5rem 0;
}

.item-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-group label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.form-group input,
.form-group textarea {
  padding: 0.625rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  font-family: inherit;
  transition: border-color 0.15s ease;
}

.form-group input:focus,
.form-group textarea:focus {
  outline: none;
  border-color: #3b82f6;
}

.form-group textarea {
  resize: vertical;
}

.form-error {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #991b1b;
  font-size: 0.875rem;
  font-weight: 500;
}

.po-view {
  min-height: 100px;
}

.view-status {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.view-status.error {
  color: #dc2626;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.5rem;
}

.info-item-full {
  grid-column: 1 / -1;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #3b82f6;
  border: 1px solid #3b82f6;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
  border-color: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
