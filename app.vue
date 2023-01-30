<template>
<div class='main-container'>
  <div class="flex-container">
   <draggable
    class="flex-container"
    v-model='cardsData'
    @start='drag = true'
    @end='drag = false'
    @change='setNewCardOrder'
   > 
    <card-item
      v-for="card in cardsData"
      :cardData="card"
      :key="card.title"
      v-show='card.visible'
    >
    </card-item>
   </draggable> 
  </div>
  <div class='drawer'>
    <a-button type="primary" @click="toggleDrawer">
      <a-icon type="setting" theme="outlined"/> 
    </a-button>
    <a-drawer
      title="List of interfaces cards"
      placement="right"
      :closable="false"
      :visible="visibleDrawer"
      @close="toggleDrawer"
    >
      <div v-for='card in cardsData' :key='card.title'>
        <a-checkbox v-model='card.visible' @change='removeCard'>
          {{ card.title }}
        </a-checkbox>  
      </div>
    </a-drawer>
  </div>
</div>
</template>
<script>
import CardItem from './components/Card.vue';
import draggable from 'vuedraggable';

export default {
  components: {
    CardItem,
    draggable,
  },

  data() {
    return {
      cardsData: [],
      configData: [],
      cpuPercentage: 100,
      lastCPUTime: null,
      visibleDrawer: false,
    }
  },

  async created() {
    this.cardsData.push(await this.getSystemData())
    this.cardsData.push(await this.getNetworkEvents())
    this.cardsData.push(await this.getSystemEvents())
    this.cardsData.push.apply(this.cardsData, await this.getInterfacesData())
    await this.updateCpuUsage()
    await this.loadConfigData()
  },

  timers: {
    updateCardsData: {time: 5000, autostart: true, immediate: true, repeat: true},
  },
  methods: {

    updateCards(oldCards, newCards) {
      return oldCards.map(oldcard => {
        const newData = newCards.find(newcard => newcard.title === oldcard.title)
        if(newData && JSON.stringify(newData) !== JSON.stringify(oldcard)) {
          return {...oldcard, ...newData}
        } else {
          return oldcard
        }
      })
    },

    async updateCardsData() {
      const newDatas = []
      newDatas.push(await this.getSystemData())
      this.cardsData = this.updateCards(this.cardsData, newDatas)
      this.loadConfigData()
    },

    async getSystemData() {
      const system = await this.$system.getInfo().then((r) => {
        return r;
      });

      const disk = await this.$system.getDiskInfo().then((r) => {
        return r;
      })

      const d = new Date(system.localtime *1000)
      const localTime = d.toLocaleString('lt-LT')

      const memPercentage = Math.floor(((system.memory.total - system.memory.free) / system.memory.total) * 100)
      const storagePercentage = Math.floor((disk.root.used / disk.root.total) * 100)

      const cardDataSystem = {
          title: 'System',
          header: {
            name: 'CPU load',
            data: this.cpuPercentage,
            datatype: 'progress-bar'
          },
          sections: [
            {
              name: 'Uptime',
              data: '%t'.format(system.uptime),
            },
            {
              name: 'Local Device Time',
              data: localTime,
            },
            { 
              name: 'Firmware version',
              data: system.release.version,
            },
            {
              name: 'Ram',
              data: memPercentage,
              datatype: 'progress-bar'
            },
            {
              name: 'Flash',
              data: storagePercentage,
              datatype: 'progress-bar'
            }
          ],
          visible: true,
          order: ''
        }
        return cardDataSystem
    },

   async updateCpuUsage () {
      await this.$rpc.call('system', 'cpu_time').then(times => {
        if (!this.lastCPUTime) {
          this.cpuPercentage = 0
          this.lastCPUTime = times
          return
        }

        const idle1 = this.lastCPUTime[3]
        const idle2 = times[3]

        let total1 = 0
        let total2 = 0

        this.lastCPUTime.forEach(t => {
          total1 += t
        })

        times.forEach(t => {
          total2 += t
        })

        this.cpuPercentage = Math.floor(((total2 - total1 - (idle2 - idle1)) / (total2 - total1)) * 100)
        this.lastCPUTime = times
      })
    },

    async getInterfacesData() {
      const interf = await this.$network.load().then(() => {
        const interfaces = this.$network.getInterfaces()
        return interfaces
      })

      const interfaceCards = []

      for(let i = 0; i < interf.length; i++) {

        if(interf[i].name && interf[i].getIPv4Addrs().length !== 0) {
          
          const cardDataNetwork = {
            header: {},
            title: interf[i].name,
            sections: [
              {
                name: 'Type',
                data: interf[i].status.device
              },
              {
                name: 'IP Address',
                data: interf[i].getIPv4Addrs().join(',')
              }
            ],
            visible: true,
            order: ''
          }
        interfaceCards.push(cardDataNetwork)
        
        } else if (interf[i].name && interf[i].getIPv4Addrs().length === 0) {
          const cardDataNetwork = {
            header: {},
            title: interf[i].name,
            sections: [
              {
                name: 'Type',
                data: interf[i].status.device
              },
              {
                name: 'IP Address',
                data: '-'
              }
            ],
            visible: true,
            order: ''
          }
        interfaceCards.push(cardDataNetwork)  
        }
      }
      return interfaceCards
    },

    async getNetworkEvents() {
      const events = await this.$log.get({table: 'NETWORK', limit: 4}).then((r) => {
        return r
      })

      const section = events.log.map(element => {
        return {
          name: new Date(element.TIME *1000).toLocaleString('lt-LT'),
          data: element.TEXT
        }  
      })

      const cardDataNetworkEvents = {
        header: {},
        title: 'Recent network events',
        sections: section,
        visible: true,
        order: ''
      }
      return cardDataNetworkEvents
    },

    async getSystemEvents() {
      const events = await this.$log.get({table: 'SYSTEM', limit: 4}).then((r) => {
        return r
      })

      const section = events.log.map(element => {
        return {
          name: new Date(element.TIME *1000).toLocaleString('lt-LT'),
          data: JSON.stringify(element.TEXT),
        }  
      })

      const cardDataSystemEvents = {
        header: {},
        title: 'Recent system events',
        sections: section,
        visible: true,
        order: ''
      }
        return cardDataSystemEvents 
    },

    async loadConfigData() {
      await this.$uci.load('card_task').then(() => {
          this.configData = this.$uci.sections('card_task', 'interface')
          
          this.configData.forEach(config => {
            this.cardsData.forEach(card => {
              card.title === config.name ? card.visible = JSON.parse(config.visible) : null
              card.title === config.name ? card.order = JSON.parse(config.order) : null
            })
          })
      })

      this.cardsData = this.cardsData.sort((or1, or2) => (or1.order > or2.order) ? 1 : (or1.order < or2.order) ? -1 : 0)
    },  

   removeCard() {
      this.configData.forEach(config => {
        this.cardsData.forEach(card => {
          if (card.title === config.name) {
            if(card.visible === true && config.visible === 'false') {
              this.$uci.set('card_task', config['.name'], 'visible', 'true')
            
            } else if(card.visible === false && config.visible === 'true') {
            this.$uci.set('card_task', config['.name'], 'visible', 'false')
            }
          }
        })
      })

      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          this.loadConfigData()
        })
      })
    },

    setNewCardOrder() {
      for(let i = 0; i < this.configData.length; i++) {
        for(let j = 0; j < this.cardsData.length; j++) {
          if(this.cardsData[j].title === this.configData[i].name && this.configData[i].order !== j.toString()) {
            this.$uci.set('card_task', this.configData[i]['.name'], 'order', j.toString())
          }
        }
      }
      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          this.loadConfigData()
        })
      })
    },

    toggleDrawer() {
      this.visibleDrawer = !this.visibleDrawer
    },

  },
}
</script>

<style>
  .main-container {
    display: flex;
  }
  .flex-container {
    display: flex;
    flex-wrap: wrap;
  }
  .ghost {
    display: flex;
    flex-wrap: wrap;
    opacity: 0.5;
    background: #c8ebfb;
  }
  .drawer {
    margin-top: 10px;
    position: absolute;
    right: 1.5em;
  }
</style>
