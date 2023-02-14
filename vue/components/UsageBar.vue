<template>
  <div class="usage">
    <small v-if="label !== ''">{{ label }}: ({{ percent }}%)</small>
    <a-progress :percent="percent" :status="status" :showInfo="false" />
  </div>
</template>

<script>
export default {
  name: 'UsageBar',
  props: {
    label: {
      type: String,
      default: ''
    },
    percent: {
      type: Number,
      default: 0
    }
  },
  computed: {
    status () {
      return this.percent > 90 ? 'exception' : 'active'
    }
  }
}
</script>

<style scoped>
.usage {
  width: 100px;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-transform: initial !important;
}
.usage small {
  text-align: center;
  color: #1890ff;
}
</style>
