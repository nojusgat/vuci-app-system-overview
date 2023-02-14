<template>
  <a-card
    v-if="Object.keys(info).length > 0"
    :title="title"
    :hoverable="true"
    :headStyle="{
      textTransform: 'uppercase',
      fontWeight: 'bold',
      height: '50px',
      margin: '0 12px',
      padding: '0',
      color: 'rgba(0, 0, 0, 0.7)',
      borderColor: 'rgba(0, 0, 0, 0.4)'
    }"
    class="card"
    size="small"
    draggable
    @dragstart="startDrag($event, title)"
    @drop="dropDrag($event, title)"
    @dragover.prevent
    @dragenter.prevent
  >
    <component v-if="extra" :is="extra.component" slot="extra" v-bind="callFunction(extra.props)" />
    <span class="body">
      <div v-for="section in sections" :key="section.id ? callFunction(section.id) : section.title" class="section">
        <div class="title">{{ callFunction(section.title) }}</div>
        <span v-if="section.components" style="display: flex; gap: 5px;">
          <component v-for="(component, index) in section.components" :is="component.type" :key="'comp' + index"
            v-bind="callFunction(component.props)" />
        </span>
        <span v-else>
          {{ callFunction(section.render) }}
        </span>
      </div>
    </span>
  </a-card>
</template>

<script>
import UsageBar from './UsageBar.vue'

export default {
  name: 'SystemCard',
  components: {
    UsageBar
  },
  data () {
    return {
      info: {},
      additionalInfo: {}
    }
  },
  props: {
    title: {
      type: String,
      default: 'Default card'
    },
    extra: {
      type: Object,
      default: null
    },
    sections: {
      type: Array,
      default: () => []
    },
    data: {
      type: Function,
      default: null
    },
    updateTimer: {
      type: Number,
      default: 0
    }
  },
  timers: {
    update: { time: 1000, repeat: true }
  },
  methods: {
    async update () {
      this.info = await this.data(this.additionalInfo)
    },
    callFunction (func) {
      if (typeof func === 'string' || func instanceof String) return func
      else return func(this)
    },
    formatDate (date) {
      return new Date(date * 1000).toLocaleString('lt-LT')
    },
    formatMemory (memory) {
      return Math.floor(((memory.total - memory.free) / memory.total) * 100) || 0
    },
    formatFlash (disk) {
      return Math.floor((disk.root.used / disk.root.total) * 100) || 0
    },
    startDrag (event, title) {
      event.dataTransfer.dropEffect = 'move'
      event.dataTransfer.effectAllowed = 'move'
      event.dataTransfer.setData('title', title)
    },
    dropDrag (event, title) {
      const droppedTitle = event.dataTransfer.getData('title')
      this.$emit('reorder', { previousTitle: title, newTitle: droppedTitle })
    }
  },
  created () {
    if (this.data === null) return

    this.$spin(true)
    this.data(this.additionalInfo)
      .then(info => {
        this.info = info
      })
      .finally(() => {
        if (this.updateTimer !== 0) {
          this.timers.update.time = this.updateTimer
          this.$timer.start('update')
        }
        this.$spin(false)
      })
  }
}
</script>

<style>
.card {
  width: 310px;
  height: 290px;
  overflow: hidden;
}
.card .body .section {
  border-bottom: 1px solid #dddddd;
  padding-bottom: 5px;
  margin-bottom: 5px;
  font-size: 11px;
}
.card .body .section .title {
  text-transform: uppercase;
  font-weight: bold;
}
.card .body .section span {
  color: #888888;
}
.card .body .section:last-child {
  border-bottom: 0px;
}
.ant-card-small > .ant-card-head > .ant-card-head-wrapper > .ant-card-head-title {
  padding: 16px 0;
}
</style>
