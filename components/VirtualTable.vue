<template lang="pug">
table.wrapper
  tr.header(:class="{ 'mobile': isMobile }")
    th.header__column(v-for="head in table.header" v-if="!isMobile || !head.desktopOnly" :key="head.value" :style="{ width: head.width }" )
      span {{ head.label }}
      sorter(v-if="head.sortable" :sort-by="head.value" :active-sort="activeSort" @change="sort")
  recycle-scroller(v-if="sortedData.length" :emit-update="true" class="scroller" :items="sortedData" :item-size="55" pageMode)
    template(v-slot="{ item }")
      slot(name="row" :item="item")
  span.no-data(v-else) No Data
</template>

<script>
import Sorter from '~/components/Sorter'

export default {
  components: { Sorter },
  props: ['table'],
  data: () => ({ sortKey: null, route: 1 }),
  computed: {
    sortedData() {
      if (!this.sortKey) return this.table.data
      const data = [...this.table.data]
      data.sort((a, b) => a[this.sortKey] > b[this.sortKey] ? -1 : 1)
      return this.route ? data.reverse() : data
    },
    activeSort() {
      return { key: this.sortKey, route: this.route }
    }
  },
  methods: {
    sort(updated) {
      if (this.sortKey == updated.key && this.route == updated.route) {
        this.sortKey = null
        this.route = null
        return
      }
      this.sortKey = updated.key
      this.route = updated.route
    }
  }
}
</script>

<style scoped>
.wrapper {
  background: var(--table-background);
  border-radius: 8px;
  width: 100%;
}

.scroller {
  border: 1px solid var(--table-background);
}

.scroller::-webkit-scrollbar {
  display: none;
}

.header {
  position: static;
  width: 100%;
  z-index: 10;
  text-align: right;
  display: flex;
  line-height: 20px;
  padding: 15px 20px;
  border-bottom: 1px solid #282828;
}

.header__column {
  display: flex;
  gap: 5px;
  align-items: center;
  font-weight: 400;
  justify-content: end;
}

.header.mobile {
  padding: 10px 10px;
}

.header.mobile .header__column {
  width: 33.33% !important;
  font-size: 11px;
}

.header__column:first-of-type {
  justify-content: start;
}

.user {
  height: 32%;
  padding: 0 12px;
  display: flex;
  align-items: center;
}

.no-data {
  width: 100%;
  color: #909399;
  font-size: 14px;
  padding: 15px;
  display: flex;
  justify-content: center;
}
</style>
