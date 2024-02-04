<script setup lang="ts">
import { ref } from 'vue'
import useI18n from '@/lang'

interface Props {
  type: 'single' | 'multi'
  title: string
  options: string[]
}

const props = withDefaults(defineProps<Props>(), {
  options: () => []
})

const emits = defineEmits(['confirm', 'cancel', 'finish'])

const selected = ref(new Set<string>())

const { t } = useI18n.global

const handleConfirm = () => {
  let res: any = Array.from(selected.value)
  if (props.type === 'single') {
    res = res[0]
  }
  emits('confirm', res)
  emits('finish')
}

const handleCancel = () => {
  emits('cancel')
  emits('finish')
}

const isSelected = (option: string) => selected.value.has(option)

const handleSelect = (option: string) => {
  if (isSelected(option)) {
    selected.value.delete(option)
  } else {
    if (props.type === 'single') selected.value.clear()
    selected.value.add(option)
  }
}
</script>

<template>
  <div class="picker">
    <Card :title="title">
      <div class="options">
        <div v-for="(o, i) in options" :key="i" @click="handleSelect(o)" class="item">
          <span>{{ o }}</span>
          <Icon
            v-show="isSelected(o)"
            style="flex-shrink: 0"
            icon="selected"
            fill="var(--primary-color)"
          />
        </div>
      </div>

      <div class="form-action">
        <Button @click="handleCancel" size="small">{{ t('common.cancel') }}</Button>
        <Button @click="handleConfirm" size="small" type="primary">
          {{ t('common.confirm') }}
        </Button>
      </div>
    </Card>
  </div>
</template>

<style lang="less" scoped>
.picker {
  min-width: 240px;
  max-width: 60%;
  background: var(--picker-bg);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  border-radius: 8px;
  .options {
    max-height: 300px;
    overflow: auto;
  }
  .item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin: 4px 0;
    line-height: 32px;
    padding: 0 8px;
    word-wrap: break-word;
    word-break: break-all;
    &:nth-child(odd) {
      background: var(--table-tr-odd-bg);
    }
    &:nth-child(even) {
      background: var(--table-tr-even-bg);
    }
  }
}
</style>