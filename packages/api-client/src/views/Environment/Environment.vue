<script setup lang="ts">
import CodeInput from '@/components/CodeInput/CodeInput.vue'
import Sidebar from '@/components/Sidebar/Sidebar.vue'
import SidebarButton from '@/components/Sidebar/SidebarButton.vue'
import SidebarList from '@/components/Sidebar/SidebarList.vue'
import SidebarListElement from '@/components/Sidebar/SidebarListElement.vue'
import ViewLayout from '@/components/ViewLayout/ViewLayout.vue'
import ViewLayoutContent from '@/components/ViewLayout/ViewLayoutContent.vue'
import ViewLayoutSection from '@/components/ViewLayout/ViewLayoutSection.vue'
import type { HotKeyEvent } from '@/libs'
import { useWorkspace } from '@/store'
import { useModal } from '@scalar/components'
import { environmentSchema } from '@scalar/oas-utils/entities/environment'
import { nanoid } from 'nanoid'
import { inject, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import EnvironmentColorModal from './EnvironmentColorModal.vue'
import EnvironmentModal from './EnvironmentModal.vue'

const router = useRouter()
const route = useRoute()
const { environments, environmentMutators, events } = useWorkspace()
const colorModal = useModal()
const environmentModal = useModal()

const activeEnvironmentID = ref<string | null>(null)
const nameInputRef = ref<HTMLInputElement | null>(null)
const isEditingName = ref(false)
const colorModalEnvironment = ref<string | null>(null)
const selectedColor = ref('')
const layout = inject<'modal' | 'web' | 'desktop'>('layout')

const parseEnvironmentValue = (value: string): Record<string, string> =>
  JSON.parse(value)

const stringifyEnvironmentValue = (value: Record<string, string>): string =>
  JSON.stringify(value, null, 2)

function addEnvironment(environment: { name: string; color: string }) {
  const existingEnvironment = environments[Object.keys(environments)[0]]
  const defaultKeys = existingEnvironment
    ? Object.keys(parseEnvironmentValue(existingEnvironment.value))
    : []

  const newEnvironmentValue = defaultKeys.reduce(
    (acc, key) => {
      acc[key] = ''
      return acc
    },
    {} as Record<string, string>,
  )

  const newEnvironment = environmentSchema.parse({
    name: environment.name,
    uid: nanoid(),
    color: environment.color,
    value: stringifyEnvironmentValue(newEnvironmentValue),
    isDefault: false,
  })

  environmentMutators.add(newEnvironment)
  activeEnvironmentID.value = newEnvironment.uid
  router.push(activeEnvironmentID.value)
  environmentModal.hide()
}

function synchronizeKeys(newKey: string) {
  Object.values(environments).forEach((env) => {
    const envValue = parseEnvironmentValue(env.value)
    if (!(newKey in envValue)) {
      envValue[newKey] = ''
      environmentMutators.edit(
        env.uid,
        'value',
        stringifyEnvironmentValue(envValue),
      )
    }
  })
}

function synchronizeKeyRemoval(removedKey: string) {
  Object.values(environments).forEach((env) => {
    const envValue = parseEnvironmentValue(env.value)
    if (removedKey in envValue) {
      delete envValue[removedKey]
      environmentMutators.edit(
        env.uid,
        'value',
        stringifyEnvironmentValue(envValue),
      )
    }
  })
}

function handleEnvironmentUpdate(raw: string) {
  if (activeEnvironmentID.value) {
    const updatedValue = parseEnvironmentValue(raw)

    const currentValue = parseEnvironmentValue(
      environments[activeEnvironmentID.value].value,
    )

    Object.keys(updatedValue).forEach((key) => {
      if (!(key in currentValue)) {
        synchronizeKeys(key)
      }
    })

    Object.keys(currentValue).forEach((key) => {
      if (!(key in updatedValue)) {
        synchronizeKeyRemoval(key)
      }
    })

    environmentMutators.edit(activeEnvironmentID.value, 'value', raw)
  }
}

const removeEnvironment = (uid: string) => {
  environmentMutators.delete(uid)

  if (activeEnvironmentID.value === uid) {
    const remainingEnvironments = Object.values(environments)

    if (remainingEnvironments.length > 0) {
      // Redirect to the last environment
      const lastEnvironment =
        remainingEnvironments[remainingEnvironments.length - 1]

      activeEnvironmentID.value = lastEnvironment.uid

      router.push({
        name: 'environment',
        params: { environment: lastEnvironment.uid },
      })
    } else {
      // Redirect to the default environment
      activeEnvironmentID.value = environments.default.uid

      router.push({ name: 'environment', params: { environment: 'default' } })
    }
  }
}

const handleOpenColorModal = (uid: string) => {
  colorModalEnvironment.value = uid
  selectedColor.value = environments[uid].color || ''
  colorModal.show()
}

const submitColorChange = (color: string) => {
  if (colorModalEnvironment.value) {
    environmentMutators.edit(colorModalEnvironment.value, 'color', color)
    colorModal.hide()
  }
}

/** set active environment based on the route */
const setActiveEnvironment = () => {
  const routeEnvironmentId = router.currentRoute.value.params
    .environment as string
  if (routeEnvironmentId) {
    activeEnvironmentID.value = routeEnvironmentId
  } else if (routeEnvironmentId === 'default') {
    activeEnvironmentID.value = environments.default.uid
  }
}

/** display a focused input to edit environment name */
const enableNameEditing = () => {
  if (
    activeEnvironmentID.value &&
    !environments[activeEnvironmentID.value].isDefault
  ) {
    isEditingName.value = true
    nextTick(() => {
      nameInputRef.value?.focus()
    })
  }
}

const updateEnvironmentName = (event: Event) => {
  const target = event.target as HTMLInputElement
  const newName = target.value
  if (
    activeEnvironmentID.value &&
    !environments[activeEnvironmentID.value].isDefault
  ) {
    environmentMutators.edit(activeEnvironmentID.value, 'name', newName)
  }
}

const handleHotKey = (event?: HotKeyEvent) => {
  if (event?.createNew && route.name === 'environment') {
    addEnvironment({ name: '', color: '' })
  }
}

const openEnvironmentModal = () => {
  environmentModal.show()
}

watch(
  () => route.params.environment,
  (newEnvironmentId) =>
    (activeEnvironmentID.value =
      (newEnvironmentId as string) || environments.default.uid),
)

onMounted(() => {
  setActiveEnvironment()
  events.hotKeys.on(handleHotKey)
})
onBeforeUnmount(() => events.hotKeys.off(handleHotKey))
</script>
<template>
  <ViewLayout>
    <Sidebar title="Environment">
      <template #content>
        <div class="flex-1">
          <SidebarList>
            <SidebarListElement
              v-for="environment in environments"
              :key="environment.uid"
              class="text-xs"
              :isCopyable="false"
              :variable="{
                name: environment.name,
                uid: environment.uid,
                color: environment.color,
                isDefault: environment.isDefault,
              }"
              :warningMessage="`Are you sure you want to delete this environment?`"
              @click="activeEnvironmentID = environment.uid"
              @colorModal="handleOpenColorModal"
              @delete="removeEnvironment(environment.uid)" />
          </SidebarList>
        </div>
      </template>
      <template #button>
        <SidebarButton
          :click="openEnvironmentModal"
          hotkey="N"
          :layout="layout">
          <template #title>Add Environment</template>
        </SidebarButton>
      </template>
    </Sidebar>
    <ViewLayoutContent class="flex-1">
      <ViewLayoutSection>
        <template
          v-if="activeEnvironmentID"
          #title>
          <span
            v-if="!isEditingName || environments[activeEnvironmentID].isDefault"
            @dblclick="enableNameEditing">
            {{ environments[activeEnvironmentID].name }}
          </span>
          <input
            v-else
            ref="nameInputRef"
            class="ring-1 ring-offset-4 ring-b-outline rounded"
            spellcheck="false"
            type="text"
            :value="environments[activeEnvironmentID].name"
            @blur="isEditingName = false"
            @input="updateEnvironmentName"
            @keyup.enter="isEditingName = false" />
        </template>
        <CodeInput
          v-if="activeEnvironmentID"
          class="pl-px pr-2 md:px-4 py-2"
          isCopyable
          language="json"
          lineNumbers
          lint
          :modelValue="environments[activeEnvironmentID].value"
          @update:modelValue="handleEnvironmentUpdate" />
      </ViewLayoutSection>
    </ViewLayoutContent>
    <EnvironmentColorModal
      :selectedColor="selectedColor"
      :state="colorModal"
      @cancel="colorModal.hide()"
      @submit="submitColorChange" />
    <EnvironmentModal
      :state="environmentModal"
      @cancel="environmentModal.hide()"
      @submit="addEnvironment" />
  </ViewLayout>
</template>
