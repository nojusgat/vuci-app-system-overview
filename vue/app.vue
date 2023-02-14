<template>
  <div>
    <div class="card-container">
      <system-card
        v-for="card in orderedCards"
        :key="card.title"
        v-bind="card"
        @reorder="swapCardOrder"
      />
    </div>
    <drawer-settings
      :cards="availableCards"
      :visible="isVisibleSettings"
      @open="openSettings"
      @close="closeSettings"
      @add="addCard"
      @remove="removeCard"
    />
  </div>
</template>

<script>
import SystemCard from './components/SystemCard.vue'
import DrawerSettings from './components/DrawerSettings.vue'

export default {
  components: {
    SystemCard,
    DrawerSettings
  },
  data () {
    return {
      availableCards: [],
      cards: [],
      networkEventCount: 4,
      systemEventCount: 4,
      isVisibleSettings: false
    }
  },
  methods: {
    getSystemData (additionalInfo) {
      return this.$system.getInfo()
        .then(info => this.updateCpuUsage(info, additionalInfo))
    },
    getInterfaceData (name) {
      return this.$network.load()
        .then(() => this.$network.getInterface(name))
    },
    getEventsData (table, limit) {
      return this.$log.get({ table: table, limit: limit })
    },
    updateCpuUsage (info, additionalInfo) {
      return this.$rpc.call('system', 'cpu_time').then(times => {
        if (!additionalInfo.lastCPUTime) {
          info.cpuPercentage = 0
          additionalInfo.lastCPUTime = times
          return info
        }

        const idle1 = additionalInfo.lastCPUTime[3]
        const idle2 = times[3]

        let total1 = 0
        let total2 = 0

        additionalInfo.lastCPUTime.forEach(t => {
          total1 += t
        })

        times.forEach(t => {
          total2 += t
        })

        info.cpuPercentage = Math.floor(((total2 - total1 - (idle2 - idle1)) / (total2 - total1)) * 100)
        additionalInfo.lastCPUTime = times
        return info
      })
    },
    swapCardOrder ({ newTitle, previousTitle }) {
      const previousCard = this.cards.findIndex(card => card.title === previousTitle)
      const newCard = this.cards.findIndex(card => card.title === newTitle)
      if (previousCard === -1 || newCard === -1) return

      const newOrder = this.cards[newCard].order
      const previousOrder = this.cards[previousCard].order

      this.cards[newCard].order = previousOrder
      this.cards[previousCard].order = newOrder

      this.$uci
        .load('system_overview')
        .then(() => {
          this.$uci.set('system_overview', 'order', newTitle.replaceAll(' ', '_'), previousOrder)
          this.$uci.set('system_overview', 'order', previousTitle.replaceAll(' ', '_'), newOrder)
        })
        .then(() => this.$uci.save())
    },
    async addCard (title) {
      const card = this.availableCards.find(card => card.title === title)
      if (!card) return

      const order = await this.getCardsOrder()
      if (title in order) {
        card.order = order[title]
      }

      this.cards.push(card)
    },
    removeCard (title) {
      this.cards = this.cards.filter(card => card.title !== title)
    },
    openSettings () {
      this.isVisibleSettings = true
    },
    closeSettings () {
      this.isVisibleSettings = false
    },
    async getCardsVisibility () {
      let cardsInfo = await this.$uci
        .load('system_overview')
        .then(() =>
          this.$uci
            .sections('system_overview')
            .filter((s) => s['.name'] === 'visibility')
        )

      if (cardsInfo.length < 1) return

      cardsInfo = Object.keys(cardsInfo[0]).reduce((acc, key) => {
        if (key.startsWith('.')) return acc
        return {
          ...acc,
          [key.replaceAll('_', ' ')]: Boolean(Number(cardsInfo[0][key]))
        }
      }, {})

      return cardsInfo
    },
    async getCardsOrder () {
      let cardsInfo = await this.$uci
        .load('system_overview')
        .then(() =>
          this.$uci
            .sections('system_overview')
            .filter((s) => s['.name'] === 'order')
        )

      if (cardsInfo.length < 1) return

      cardsInfo = Object.keys(cardsInfo[0]).reduce((acc, key) => {
        if (key.startsWith('.')) return acc
        return {
          ...acc,
          [key.replaceAll('_', ' ')]: Number(cardsInfo[0][key])
        }
      }, {})

      return cardsInfo
    },
    async getAvailableCards () {
      const cardsInfo = await this.$uci
        .load('system_overview')
        .then(() =>
          this.$uci
            .sections('system_overview')
            .filter((s) => s['.name'] === 'config')
        )

      if (cardsInfo.length < 1) return

      return cardsInfo[0].cards
    }
  },
  computed: {
    orderedCards () {
      return [...this.cards].sort((a, b) => a.order - b.order)
    },
    systemCard () {
      return {
        title: 'System',
        extra: {
          component: 'usage-bar',
          props: card => ({ label: 'CPU load', percent: card.info.cpuPercentage || 0 })
        },
        sections: [
          {
            title: 'Router uptime',
            render: card => '%t'.format(card.info.uptime)
          },
          {
            title: 'Local device time',
            render: card => card.formatDate(card.info.localtime)
          },
          {
            title: 'Memory usage',
            components: [
              {
                type: 'usage-bar',
                props: card => ({ label: 'RAM', percent: card.formatMemory(card.info.memory) })
              },
              {
                type: 'usage-bar',
                props: card => ({ label: 'FLASH', percent: card.formatFlash(card.info.disk) })
              }
            ]
          },
          {
            title: 'Firmware version',
            render: card => card.info.release.revision || ''
          }
        ],
        data: this.getSystemData,
        updateTimer: 1000,
        order: 0
      }
    },
    lanCard () {
      return {
        title: 'Lan',
        sections: [
          {
            title: 'Type',
            render: card => card.info.status.l3_device || 'offline'
          },
          {
            title: 'IP address',
            render: card => card.info.getIPv4Addrs(true)[0] || 'offline'
          }
        ],
        data: () => this.getInterfaceData('lan'),
        order: 1
      }
    },
    wanCard () {
      return {
        title: 'Wan',
        sections: [
          {
            title: 'Type',
            render: card => card.info.status.l3_device || 'offline'
          },
          {
            title: 'IP address',
            render: card => card.info.getIPv4Addrs(true)[0] || 'offline'
          }
        ],
        data: () => this.getInterfaceData('wan'),
        order: 2
      }
    },
    networkEventsCard () {
      return {
        title: 'Recent network events',
        sections: Array.from(Array(this.networkEventCount).keys()).map(val => {
          return {
            id: card => card.info.log[val].ID,
            title: card => card.formatDate(card.info.log[val].TIME),
            render: card => card.info.log[val].TEXT
          }
        }),
        data: () => this.getEventsData('NETWORK', this.networkEventCount),
        order: 3
      }
    },
    systemEventsCard () {
      return {
        title: 'Recent system events',
        sections: Array.from(Array(this.systemEventCount).keys()).map(val => {
          return {
            id: card => card.info.log[val].ID,
            title: card => card.formatDate(card.info.log[val].TIME),
            render: card => card.info.log[val].TEXT
          }
        }),
        data: () => this.getEventsData('SYSTEM', this.systemEventCount),
        order: 4
      }
    }
  },
  async created () {
    const availableCards = await this.getAvailableCards()
    const visibility = await this.getCardsVisibility()
    const order = await this.getCardsOrder()

    this.availableCards = availableCards.map(card => this[card])

    this.availableCards.forEach((card) => {
      if (visibility[card.title]) {
        if (card.title in order) {
          card.order = order[card.title]
        }
        this.cards.push(card)
      }
    })
  }
}
</script>

<style scoped>
.card-container {
  display: flex;
  gap: 10px 10px;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: center;
}
</style>
