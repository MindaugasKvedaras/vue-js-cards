<template>
  <a-card style='width: 320px; height: 360px; margin: 10px; cursor: grab'>
    <div class="card-title">
      <h1>{{ card.title.toUpperCase() }}</h1>
      <div>
        <h5>{{ card.header.name }}</h5>
        <a-progress
          v-if="card.header.datatype === 'progress-bar'"
          :percent="card.header.data"
          size='small'
          status='active'
        />
      </div>
    </div>
    <a-divider style='height: 1px; margin: 0; background-color: #605D5C' />
    <div
      class='section'
      v-for='(item, index) in card.sections'
      :key='index'
    >
      <div>
        <h4>{{ item.name.toUpperCase() }}</h4>
      </div>
      <div>
        <h5 v-if="item.datatype != 'progress-bar'">{{ item.data }}</h5>
        <a-progress
          v-if="item.datatype === 'progress-bar'"
          :percent="item.data"
          size='small'
          status='active'
        />
      </div>
      <a-divider style='margin: 0' />
    </div>
  </a-card>
</template>

<script>
export default {
  name: 'CardItem',
  data () {
    return {
      card: this.cardData
    }
  },
  props: {
    cardData: {
      type: Object
    }
  },
  watch: {
    cardData: {
      handler: function (newVal) {
        this.card = newVal
      },
      deep: true
    }
  }
}
</script>

<style>
.card-title {
  display: flex;
  justify-content: space-between;
}
.card-title h5 {
  margin-bottom: -0.6em;
}
.section h5 {
  color: grey;
}
</style>
