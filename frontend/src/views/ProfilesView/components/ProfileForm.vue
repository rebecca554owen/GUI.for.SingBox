<script setup lang="ts">
import { ref, inject, type Ref, computed } from 'vue'
import { useI18n } from 'vue-i18n'

import { useMessage, useAlert, useBool } from '@/hooks'
import { deepClone, generateConfig, sampleID } from '@/utils'
import * as Defaults from '@/constant/profile'
import { WindowToggleMaximise } from '@/bridge'
import { type ProfileType, useProfilesStore } from '@/stores'

import GeneralConfig from './GeneralConfig.vue'
import AdvancedConfig from './AdvancedConfig.vue'
import TunConfig from './TunConfig.vue'
import DnsConfig from './DnsConfig.vue'
import ProxyGroupsConfig from './ProxyGroupsConfig.vue'
import DnsRulesConfig from './DnsRulesConfig.vue'
import RulesConfig from './RulesConfig.vue'
import MixinAndScript from './MixinAndScriptConfig.vue'

interface Props {
  id?: string
  step?: number
  isUpdate?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  id: '',
  isUpdate: false,
  step: 0
})

const loading = ref(false)
const groupsRef = ref()
const rulesRef = ref()
const dnsRulesRef = ref()
const currentStep = ref(props.step)

const stepItems = [
  { title: 'profile.step.name' },
  { title: 'profile.step.general' },
  { title: 'profile.step.tun' },
  { title: 'profile.step.groups' },
  { title: 'profile.step.rules' },
  { title: 'profile.step.dns' },
  { title: 'profile.step.dnsRules' },
  { title: 'profile.step.mixin-script' }
]

const profile = ref<ProfileType>({
  id: sampleID(),
  name: '',
  generalConfig: Defaults.GeneralConfigDefaults(),
  advancedConfig: Defaults.AdvancedConfigDefaults(),
  tunConfig: Defaults.TunConfigDefaults(),
  dnsConfig: Defaults.DnsConfigDefaults(),
  proxyGroupsConfig: Defaults.ProxyGroupsConfigDefaults(),
  rulesConfig: Defaults.RulesConfigDefaults(),
  dnsRulesConfig: Defaults.DnsRulesConfigDefaults(),
  mixinConfig: Defaults.MixinConfigDefaults(),
  scriptConfig: Defaults.ScriptConfigDefaults()
})

const mixinAndScriptConfig = computed({
  get() {
    return { mixin: profile.value.mixinConfig, script: profile.value.scriptConfig }
  },
  set({ mixin, script }) {
    profile.value.mixinConfig = mixin
    profile.value.scriptConfig = script
  }
})

const { t } = useI18n()
const { alert } = useAlert()
const { message } = useMessage()
const profilesStore = useProfilesStore()
const [showAdvancedSetting, toggleAdvancedSetting] = useBool(false)

const handleCancel = inject('cancel') as any
const handleSubmit = inject('submit') as any
const handlePrevStep = () => currentStep.value--
const handleNextStep = () => currentStep.value++

const handleSave = async () => {
  loading.value = true
  try {
    if (props.isUpdate) {
      await profilesStore.editProfile(props.id, profile.value)
      handleSubmit()
    } else {
      await profilesStore.addProfile(profile.value)
      handleCancel()
    }
  } catch (error: any) {
    console.error('handleSave: ', error)
    message.error(error)
  }
  loading.value = false
}

const handleAdd = () => {
  const map: Record<number, Ref> = {
    '3': groupsRef,
    '4': rulesRef,
    '6': dnsRulesRef
  }
  map[currentStep.value].value.handleAdd()
}

const handlePreview = async () => {
  try {
    const config = await generateConfig(profile.value)
    alert(profile.value.name, JSON.stringify(config, null, 2))
  } catch (error: any) {
    message.error(error.message || error)
  }
}

if (props.isUpdate) {
  const p = profilesStore.getProfileById(props.id)
  if (p) {
    profile.value = deepClone(p)
  }
}
</script>

<template>
  <div @dblclick="WindowToggleMaximise" class="header" style="--wails-draggable: drag">
    <div class="header-title">{{ t(stepItems[currentStep].title) }}</div>
    <Button @click="handlePreview" icon="file" type="text" class="ml-auto" />
    <Button
      v-show="[3, 4, 6].includes(currentStep)"
      @click="handleAdd"
      icon="add"
      type="text"
      class="mr-8"
    />
  </div>

  <div class="form">
    <div v-show="currentStep === 0">
      <div class="form-item">
        <div class="name">{{ t('profile.name') }} *</div>
        <Input v-model="profile.name" auto-size autofocus class="flex-1 ml-8" />
      </div>
    </div>

    <div v-show="currentStep === 1">
      <GeneralConfig v-model="profile.generalConfig" />
      <Divider>
        <Button type="text" size="small" @click="toggleAdvancedSetting">
          {{ t('profile.advancedSettings') }}
        </Button>
      </Divider>
      <div v-if="showAdvancedSetting">
        <AdvancedConfig v-model="profile.advancedConfig" />
      </div>
    </div>

    <div v-show="currentStep === 2">
      <TunConfig v-model="profile.tunConfig" />
    </div>

    <div v-show="currentStep === 3">
      <ProxyGroupsConfig ref="groupsRef" v-model="profile.proxyGroupsConfig" />
    </div>

    <div v-show="currentStep === 4">
      <RulesConfig
        ref="rulesRef"
        v-model="profile.rulesConfig"
        :proxy-groups="profile.proxyGroupsConfig"
        :profile="profile"
      />
    </div>

    <div v-show="currentStep === 5">
      <DnsConfig v-model="profile.dnsConfig" :proxy-groups="profile.proxyGroupsConfig" />
    </div>

    <div v-show="currentStep === 6">
      <DnsRulesConfig
        ref="dnsRulesRef"
        v-model="profile.dnsRulesConfig"
        :dns-config="profile.dnsConfig"
        :proxy-groups="profile.proxyGroupsConfig"
      />
    </div>

    <div v-show="currentStep === 7">
      <MixinAndScript v-model="mixinAndScriptConfig" />
    </div>
  </div>

  <div class="form-action">
    <Button @click="handlePrevStep" :disabled="currentStep == 0" type="text">
      {{ t('common.prevStep') }}
    </Button>
    <Button
      @click="handleNextStep"
      :disabled="!profile.name || currentStep == stepItems.length - 1"
      type="text"
      class="mr-auto"
    >
      {{ t('common.nextStep') }}
    </Button>
    <Button @click="handleCancel">{{ t('common.cancel') }}</Button>
    <Button @click="handleSave" :loading="loading" :disabled="!profile.name" type="primary">
      {{ t('common.save') }}
    </Button>
  </div>
</template>

<style lang="less" scoped>
.header {
  display: flex;
  align-items: center;
  margin-top: 8px;
  &-title {
    font-size: 20px;
    font-weight: bold;
    margin: 8px 0 16px 0;
  }
}
.form {
  padding-right: 8px;
  overflow-y: auto;
  max-height: calc(70vh - 8px);
}
</style>
