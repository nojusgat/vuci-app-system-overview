<template>
  <div>
    <a-button class="settings-icon" type="primary" icon="setting" size="large" @click="$emit('open')" />
    <a-drawer
      title="Settings"
      placement="right"
      :closable="true"
      :visible="visible"
      :after-visible-change="loadData"
      @close="$emit('close')"
    >
      <vuci-form uci-config="system_overview" ref="form">
        <vuci-named-section name="visibility" v-slot="{ s }" :card="false">
          <vuci-form-item-switch
            class="settings-item"
            v-for="card in cards"
            :key="card.title"
            :label="card.title"
            :uci-section="s"
            :name="convertTitle(card.title)"
            :initial="false"
            true-value="1"
            false-value="0"
            @change="change"
          />
        </vuci-named-section >
        <template #footer>
          <div></div>
        </template>
      </vuci-form>
    </a-drawer>
  </div>
</template>

<script>
export default {
  name: 'DrawerSettings',
  props: {
    visible: {
      type: Boolean,
      default: false
    },
    cards: {
      type: Array,
      default: () => []
    }
  },
  methods: {
    loadData (visible) {
      if (visible) this.$refs.form.load()
    },
    change (input) {
      this.$refs.form.apply()
      if (!input.model) return this.$emit('remove', input.label)
      this.$emit('add', input.label)
    },
    convertTitle (title) {
      return title.replaceAll(' ', '_')
    }
  }
}
</script>

<style >
.settings-icon {
  position: fixed;
  top: 60px;
  right: -5px;
  width: 50px;
}
.settings-item {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  justify-content: center;
}
.settings-item .ant-form-item-label {
  text-align: center;
  width: 100%;
  line-height: initial;
}
.settings-item .ant-col-14 {
  width: initial;
}
</style>
