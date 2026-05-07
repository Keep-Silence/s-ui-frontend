<template>
  <v-dialog
    v-model="internalVisible"
    transition="dialog-bottom-transition"
    max-width="500"
    scrollable
  >
    <v-form @submit.prevent="save">
      <v-card rounded="xl" elevation="5" :title="(id ?? 0) > 0 ? $t('actions.edit') + ' ' + $t('objects.node') : $t('actions.add') + ' ' + $t('objects.node')">
        <v-card-text>
          <v-text-field
            v-model="node.name"
            :label="$t('node.name')"
            density="comfortable"
            required
          ></v-text-field>
          <v-text-field
            v-model="node.addr"
            :label="$t('node.addr')"
            density="comfortable"
            required
          ></v-text-field>
          <v-text-field
            v-model="node.token"
            :label="$t('node.token')"
            density="comfortable"
            required
          ></v-text-field>
        </v-card-text>
        <v-divider></v-divider>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn
            color="primary"
            variant="outlined"
            @click="close"
          >
            {{ $t('actions.close') }}
          </v-btn>
          <v-btn
            color="primary"
            variant="tonal"
            :loading="loading"
            type="submit"
          >
            {{ $t('actions.save') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-form>
  </v-dialog>
</template>

<script lang="ts" setup>
import { computed, ref, watch } from 'vue'
import Data from '@/store/modules/data'

interface NodeData {
  id?: number
  addr: string
  name: string
  token: string
  last_heartbeat?: number
}

const props = defineProps({
  visible: Boolean,
  id: Number,
  data: String,
})

const emit = defineEmits(['update:visible', 'close'])

const internalVisible = computed({
  get: () => props.visible,
  set: (val) => emit('update:visible', val)
})

const loading = ref(false)
const node = ref<NodeData>({
  addr: "",
  name: "",
  token: ""
})

watch(() => props.visible, (val) => {
  if (val) {
    if (props.id && props.id > 0 && props.data) {
      node.value = JSON.parse(props.data)
    } else {
      node.value = {
        addr: "",
        name: "",
        token: Math.random().toString(36).substring(2, 15) // Generate random token
      }
    }
  }
})

const close = () => {
  emit('close')
}

const save = async () => {
  const isNew = (!props.id || props.id === 0)
  const action = isNew ? "new" : "edit"

  if (node.value.name === '' || node.value.token === '') {
    return
  }

  loading.value = true
  const success = await Data().save("nodes", action, node.value)
  if (success) {
    close()
  }
  loading.value = false
}
</script>
