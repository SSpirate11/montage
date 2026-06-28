<template>
  <cdx-dialog
    :open="open"
    @update:open="(value) => emit('update:open', value)"
    :title="$t('montage-flag-dialog-title')"
    :use-close-button="true"
    :primary-action="primaryAction"
    :default-action="{ label: $t('montage-btn-cancel') }"
    @primary="submitFlag"
    @default="emit('update:open', false)"
  >
    <p class="flag-dialog-help">{{ $t('montage-flag-dialog-help') }}</p>
    <div class="flag-category-buttons">
      <cdx-button
        v-for="cat in categories"
        :key="cat.key"
        :action="selectedCategory === cat.key ? 'progressive' : 'default'"
        :weight="selectedCategory === cat.key ? 'primary' : 'normal'"
        @click="selectedCategory = cat.key"
      >
        {{ $t(cat.label) }}
      </cdx-button>
    </div>
    <div v-if="selectedCategory" class="flag-note">
      <label class="flag-note-label">
        {{ $t('montage-flag-note-label') }}
        <span class="greyed">
          ({{
            selectedCategory === 'other'
              ? $t('montage-flag-note-required')
              : $t('montage-flag-note-optional')
          }})
        </span>
      </label>
      <cdx-text-area v-model="note" :placeholder="$t('montage-flag-note-placeholder')" />
    </div>
  </cdx-dialog>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { CdxDialog, CdxButton, CdxTextArea } from '@wikimedia/codex'
import jurorService from '@/services/jurorService'
import alertService from '@/services/alertService'

const { t: $t } = useI18n()

const props = defineProps({
  open: Boolean,
  roundId: [Number, String],
  entryId: [Number, String]
})

const emit = defineEmits(['update:open', 'flagged'])

// Stable category keys (stored as-is); labels are translated for display only.
const categories = [
  { key: 'copyright', label: 'montage-flag-category-copyright' },
  { key: 'scope', label: 'montage-flag-category-scope' },
  { key: 'authorship', label: 'montage-flag-category-authorship' },
  { key: 'quality', label: 'montage-flag-category-quality' },
  { key: 'ai', label: 'montage-flag-category-ai' },
  { key: 'other', label: 'montage-flag-category-other' }
]

const selectedCategory = ref(null)
const note = ref('')

// Note is optional, except when the catch-all "other" category is chosen.
const canSubmit = computed(() => {
  if (!selectedCategory.value) return false
  if (selectedCategory.value === 'other') return note.value.trim().length > 0
  return true
})

const primaryAction = computed(() => ({
  label: $t('montage-flag-submit'),
  actionType: 'progressive',
  disabled: !canSubmit.value
}))

// Reset the form whenever the dialog is (re)opened.
watch(
  () => props.open,
  (isOpen) => {
    if (isOpen) {
      selectedCategory.value = null
      note.value = ''
    }
  }
)

const submitFlag = () => {
  if (!canSubmit.value) return
  jurorService
    .flagImage(props.roundId, props.entryId, selectedCategory.value, note.value.trim() || null)
    .then(() => {
      alertService.success($t('montage-flag-success'), 500)
      emit('flagged')
    })
    .catch(alertService.error)
    .finally(() => {
      emit('update:open', false)
    })
}
</script>

<style scoped>
.flag-dialog-help {
  margin-bottom: 12px;
}

.flag-category-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.flag-note {
  margin-top: 16px;
}

.flag-note-label {
  display: block;
  margin-bottom: 4px;
  font-weight: 500;
}
</style>
