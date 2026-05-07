<template>
  <NodeVue
    v-model="modal.visible"
    :visible="modal.visible"
    :id="modal.id"
    :data="modal.data"
    @close="closeModal"
  />
  <Stats
    v-model="stats.visible"
    :visible="stats.visible"
    :resource="stats.resource"
    :tag="stats.tag"
    @close="closeStats"
  />
  <v-row>
    <v-col cols="12" justify="center" align="center">
      <v-btn color="primary" @click="showModal(0)">{{ $t('actions.add') }}</v-btn>
    </v-col>
  </v-row>
  <v-row>
    <v-col cols="12" sm="4" md="3" lg="2" v-for="(item, index) in <any[]>nodes" :key="item.id">
      <v-card rounded="xl" elevation="5" min-width="200" :title="item.name">
        <v-card-text>
          <v-row>
            <v-col>{{ $t('node.addr') }}</v-col>
            <v-col>{{ item.addr }}</v-col>
          </v-row>
          <v-row>
            <v-col>{{ $t('node.token') }}</v-col>
            <v-col>{{ item.token }}</v-col>
          </v-row>
          <v-row>
            <v-col>{{ $t('main.info.cpu') }}</v-col>
            <v-col>{{ item.cpu ? item.cpu.toFixed(1) + '%' : '-' }}</v-col>
          </v-row>
          <v-row>
            <v-col>{{ $t('main.info.memory') }}</v-col>
            <v-col>{{ item.mem ? item.mem.toFixed(1) + '%' : '-' }}</v-col>
          </v-row>
          <v-row>
            <v-col>{{ $t('main.info.uptime') }}</v-col>
            <v-col>{{ item.uptime ? formatUptime(item.uptime) : '-' }}</v-col>
          </v-row>
          <v-row>
            <v-col>{{ $t('online') }}</v-col>
            <v-col>
              <template v-if="onlines.includes(item.name)">
                <v-chip density="comfortable" size="small" color="success" variant="flat">{{ $t('online') }}</v-chip>
              </template>
              <template v-else>-</template>
            </v-col>
          </v-row>
        </v-card-text>
        <v-divider></v-divider>
        <v-card-actions style="padding: 0;">
          <v-btn icon="mdi-file-edit" @click="showModal(item.id)">
            <v-icon />
            <v-tooltip activator="parent" location="top" :text="$t('actions.edit')"></v-tooltip>
          </v-btn>
          <v-btn v-if="nodeInbounds(item.id).length === 0" icon="mdi-file-remove" style="margin-inline-start:0;" color="warning" @click="delOverlay[index] = true">
            <v-icon />
            <v-tooltip activator="parent" location="top" :text="$t('actions.del')"></v-tooltip>
          </v-btn>
          <v-overlay
            v-model="delOverlay[index]"
            contained
            class="align-center justify-center"
          >
            <v-card :title="$t('actions.del')" rounded="lg">
              <v-divider></v-divider>
              <v-card-text>{{ $t('confirm') }}</v-card-text>
              <v-card-actions>
                <v-btn color="error" variant="outlined" @click="delNode(item.id)">{{ $t('yes') }}</v-btn>
                <v-btn color="success" variant="outlined" @click="delOverlay[index] = false">{{ $t('no') }}</v-btn>
              </v-card-actions>
            </v-card>
          </v-overlay>
          <v-btn icon="mdi-chart-line" @click="showStats(item.name)" v-if="Data().enableTraffic">
            <v-icon />
            <v-tooltip activator="parent" location="top" :text="$t('stats.graphTitle')"></v-tooltip>
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-col>
  </v-row>
</template>

<script lang="ts" setup>
import Data from '@/store/modules/data'
import NodeVue from '@/layouts/modals/Node.vue'
import Stats from '@/layouts/modals/Stats.vue'
import { computed, ref } from 'vue'

const nodes = computed((): any[] => {
  return Data().nodes
})

const onlines = computed(() => {
  return Data().onlines.node?? []
})

const nodeInbounds = (id: number) => {
  return (Data().inbounds ?? []).filter(i => i.nodes.includes(id))
}

const modal = ref({
  visible: false,
  id: 0,
  data: "",
})

let delOverlay = ref(new Array<boolean>)

const showModal = (id: number) => {
  modal.value.id = id
  modal.value.data = id == 0 ? '' : JSON.stringify(nodes.value.findLast(o => o.id == id))
  modal.value.visible = true
}

const closeModal = () => {
  modal.value.visible = false
}

const formatUptime = (uptime: number) => {
  const d = Math.floor(uptime / 86400)
  const h = Math.floor((uptime % 86400) / 3600)
  const m = Math.floor((uptime % 3600) / 60)
  const s = Math.floor(uptime % 60)
  if (d > 0) return `${d}d ${h}h ${m}m`
  return `${d > 0 ? d + 'd ' : ''}${h > 0 ? h + 'h ' : ''}${m > 0 ? m + 'm ' : ''}${s}s`
}

const stats = ref({
  visible: false,
  resource: "node",
  tag: "",
})

const delNode = async (id: number) => {
  const index = nodes.value.findIndex(i => i.id == id)
  const success = await Data().save("nodes", "del", id)
  if (success) delOverlay.value[index] = false
}

const showStats = (tag: string) => {
  stats.value.tag = tag
  stats.value.visible = true
}
const closeStats = () => {
  stats.value.visible = false
}
</script>
