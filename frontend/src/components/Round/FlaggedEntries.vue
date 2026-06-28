<template>
  <div class="flagged-entries">
    <h3>{{ $t('montage-round-flagged-images') }}</h3>
    <p v-if="loading" class="greyed">{{ $t('montage-loading') }}</p>
    <p v-else-if="!entries.length" class="greyed">{{ $t('montage-round-no-flagged-images') }}</p>
    <div v-else>
      <p class="greyed">{{ $t('montage-round-flagged-images-help') }}</p>
      <div
        v-for="entry in entries"
        :key="entry.id"
        class="flagged-entry"
        :class="{ 'is-disqualified': entry.dq_done }"
      >
        <div class="flagged-entry-thumb">
          <CommonsImage :image="entry" :width="160" />
        </div>
        <div class="flagged-entry-body">
          <div class="flagged-entry-header">
            <a
              :href="'https://commons.wikimedia.org/wiki/File:' + entry.name"
              target="_blank"
              class="flagged-entry-name"
            >
              {{ entry.name.split('_').join(' ') }}
            </a>
            <span class="flagged-entry-count">
              {{ $t('montage-round-flag-count', [entry.flaggings.length]) }}
            </span>
          </div>
          <ul class="flagged-entry-list">
            <li v-for="(flag, index) in entry.flaggings" :key="index">
              <span class="flag-category">{{ categoryLabel(flag.category) }}</span>
              <span class="greyed"> · {{ flag.user }} · {{ flag.date }}</span>
              <div v-if="flag.reason" class="flag-reason">{{ flag.reason }}</div>
            </li>
          </ul>
        </div>
        <div class="flagged-entry-actions">
          <cdx-button
            action="destructive"
            weight="primary"
            :disabled="entry.dq_done || !canDisqualify"
            @click="disqualify(entry)"
          >
            <close class="icon-small" />
            {{ entry.dq_done ? $t('montage-round-disqualified') : $t('montage-round-disqualify') }}
          </cdx-button>
          <p v-if="!canDisqualify && !entry.dq_done" class="greyed disqualify-hint">
            {{ $t('montage-round-disqualify-needs-pause') }}
          </p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useI18n } from 'vue-i18n'
import { CdxButton } from '@wikimedia/codex'
import adminService from '@/services/adminService'
import alertService from '@/services/alertService'
import CommonsImage from '@/components/CommonsImage.vue'
import Close from 'vue-material-design-icons/Close.vue'

const { t: $t } = useI18n()

const props = defineProps({
  roundId: [Number, String],
  roundStatus: String
})

// The backend only allows manual disqualification on a paused round.
const canDisqualify = computed(() => props.roundStatus === 'paused')

const entries = ref([])
const loading = ref(true)

const categoryLabel = (category) => {
  if (!category) return $t('montage-flag-category-other')
  const key = `montage-flag-category-${category}`
  const label = $t(key)
  // Fall back to the raw key if no translation exists for this category.
  return label === key ? category : label
}

const load = () => {
  loading.value = true
  adminService
    .getRoundFlags(props.roundId)
    .then((response) => {
      entries.value = response.data
    })
    .catch(alertService.error)
    .finally(() => {
      loading.value = false
    })
}

const disqualify = (entry) => {
  const categories = [...new Set(entry.flaggings.map((f) => categoryLabel(f.category)))].join(', ')
  if (!confirm($t('montage-round-disqualify-confirm', [entry.name.split('_').join(' ')]))) {
    return
  }
  adminService
    .disqualifyImage(props.roundId, entry.id, $t('montage-round-disqualify-reason', [categories]))
    .then(() => {
      entry.dq_done = true
      alertService.success($t('montage-round-disqualified'))
    })
    .catch(alertService.error)
}

defineExpose({ load, entries })

onMounted(load)
</script>

<style scoped>
.flagged-entries {
  margin-top: 24px;
}

.flagged-entry {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  padding: 12px 0;
  border-top: 1px solid #eaecf0;
}

.flagged-entry.is-disqualified {
  opacity: 0.55;
}

.flagged-entry-thumb {
  width: 160px;
  flex-shrink: 0;
}

.flagged-entry-thumb img {
  max-width: 160px;
  max-height: 120px;
  object-fit: contain;
}

.flagged-entry-body {
  flex: 1;
  min-width: 0;
}

.flagged-entry-header {
  display: flex;
  align-items: center;
  gap: 12px;
}

.flagged-entry-name {
  font-weight: 600;
  word-break: break-word;
}

.flagged-entry-count {
  color: #f9a825;
  font-weight: 600;
  white-space: nowrap;
}

.flagged-entry-list {
  margin: 8px 0 0;
  padding-left: 18px;
}

.flag-category {
  font-weight: 500;
  text-transform: capitalize;
}

.flag-reason {
  font-size: 14px;
  color: rgba(0, 0, 0, 0.7);
}

.flagged-entry-actions {
  flex-shrink: 0;
  text-align: right;
}

.disqualify-hint {
  font-size: 12px;
  margin-top: 4px;
  max-width: 160px;
}
</style>
