<template lang="pug">
.markets
  .table-intro
    el-radio-group.radio-chain-select.custom-radio(
      v-model='markets_active_tab',
      size='small'
    ).mr-3
      el-radio-button(label='fav')
        i.el-icon-star-on
        span Fav

      el-radio-button(label='all')
        span All

      el-radio-button(:label='network.baseToken.symbol')
        span {{ network.baseToken.symbol }}

      el-radio-button(v-if='network.name == "eos"', label='USDT')
        span USDT

      el-radio-button(value='cross-chain', label='Cross-Chain')
        span Cross-Chain

    .search-container
      el-input(
        v-model='search',
        placeholder='Search market',
        size='small',
        prefix-icon='el-icon-search'
        clearable
      )

    el-switch(v-if="markets_active_tab == network.baseToken.symbol" v-model='showVolumeInUSD' active-text='USD').ml-auto

    .ml-auto(v-if="!isMobile")
      nuxt-link(to="new_market")
        el-button(tag="el-button" size="small" icon="el-icon-circle-plus-outline") Open new market

  virtual-table(:table="virtualTableData")
    template(#row="{ item }")
      market-row(:item="item" :showVolumeInUSD="showVolumeInUSD" :marketsActiveTab="markets_active_tab")

  //.table.el-card.is-always-shadow
    el-table.market-table(
      :data='lazyMarkets',
      row-key="id"
      style='width: 100%',
      @row-click='clickOrder')
      el-table-column(label='Pair', prop='date')
        template(slot-scope='scope')
          TokenImage(
            :src='$tokenLogo(scope.row.quote_token.symbol.name, scope.row.quote_token.contract)',
            :height="isMobile ? '20' : '30'"
          )

          span.ml-2
            | {{ scope.row.quote_token.symbol.name }}
            span.text-muted.ml-2(v-if='!isMobile') {{ scope.row.quote_token.contract }}
            |  / {{ scope.row.base_token.symbol.name }}

          span.promoted(v-if="scope.row.promoted")
            img(src="~/assets/icons/badge-promoted.svg")

      el-table-column(
        :label='`Last price`',
        sort-by='last_price',
        align='right',
        :width='isMobile ? 90 : 150',
        header-align='right',
        sortable,
        :sort-orders='["descending", null]'
      )
        template(slot-scope='scope')
          .text-success(v-if="showVolumeInUSD && markets_active_tab == network.baseToken.symbol") ${{ $systemToUSD(scope.row.last_price, 8) }}
          .text-success(v-else) {{ scope.row.last_price }} {{ !isMobile ? scope.row.base_token.symbol.name : "" }}
      el-table-column(
        :label='`24H Vol.`',
        align='right',
        header-align='right',
        sortable,
        width='200',
        sort-by='volume24',
        :sort-orders='["descending", null]'
        v-if='!isMobile'
      )
        template(slot-scope='scope')
          span.text-mutted(v-if="showVolumeInUSD && markets_active_tab == network.baseToken.symbol") ${{ $systemToUSD(scope.row.volume24) }}
          span.text-mutted(v-else) {{ scope.row.volume24.toFixed(2) | commaFloat }} {{ scope.row.base_token.symbol.name }}

      el-table-column(
        label='24H',
        prop='name',
        align='right',
        header-align='right',
        sortable,
        width='100',
        sort-by='change24',
        :sort-orders='["descending", null]',
        v-if='!isMobile'
      )
        template(slot-scope='scope', align='right', header-align='right')
          change-percent(:change='scope.row.change24')

      el-table-column(
        label='7D Volume',
        prop='weekVolume',
        align='right',
        header-align='right',
        sortable,
        :width='isMobile ? 130 : 200',
        sort-by='volumeWeek',
        :sort-orders='["descending", null]',
      )
        template(slot-scope='scope')
          span.text-mutted(v-if="showVolumeInUSD && markets_active_tab == network.baseToken.symbol") ${{ $systemToUSD(scope.row.volumeWeek) }}
          span.text-mutted(v-else) {{ scope.row.volumeWeek.toFixed(2) | commaFloat }} {{ scope.row.base_token.symbol.name }}

      el-table-column(
        label='7D Change',
        prop='weekChange',
        align='right',
        header-align='right',
        sortable,
        width='150',
        sort-by='changeWeek',
        :sort-orders='["descending", null]',
        v-if='!isMobile'
      )
        template(slot-scope='scope')
          change-percent(:change='scope.row.changeWeek')


      template(slot="append")
        infinite-loading(@infinite='lazyloadMarkets' force-use-infinite-wrapper=".market-table.el-table__body-wrapper" spinner="spiral" ref="infinite" :identifier="skip")

        //infinite-loading(@infinite='lazyloadMarkets' spinner="spiral" ref="infinite")
</template>

<script>
import { mapState } from 'vuex'
import VirtualTable from '~/components/VirtualTable'
import MarketRow from '~/components/MarketRow'

export default {
  components: {
    VirtualTable,
    MarketRow
  },

  async fetch({ store, error }) {
    if (store.state.markets.length == 0) {
      try {
        await store.dispatch('loadMarkets')
      } catch (e) {
        return error({ message: e, statusCode: 500 })
      }
    }
  },

  data() {
    return {
      search: ''
    }
  },

  computed: {
    ...mapState(['network']),
    ...mapState(['markets']),
    ...mapState('settings', ['favMarkets']),

    markets_active_tab: {
      get() {
        return this.$store.state.market.markets_active_tab || this.network.baseToken.symbol
      },

      set(value) {
        this.$store.commit('market/setMarketActiveTab', value)
      }
    },

    showVolumeInUSD: {
      get() {
        return this.$store.state.market.showVolumeInUSD
      },

      set(value) {
        this.$store.commit('market/setShowVolumeInUSD', value)
      }
    },

    virtualTableData() {
      const header = [
        {
          label: 'Pair',
          value: 'quote_name',
          width: '340px'
        },
        {
          label: 'Last price',
          value: 'last_price',
          width: '155px',
          sortable: true
        },
        {
          label: '24H Vol.',
          value: 'volume24',
          width: '165px',
          sortable: true,
          desktopOnly: true
        },
        {
          label: '24H',
          value: 'change24',
          width: '80px',
          sortable: true,
          desktopOnly: true
        },
        {
          label: '7D Volume',
          value: 'volume_week',
          width: '180px',
          sortable: true
        },
        {
          label: '7D Change',
          value: 'change_week',
          width: '165px',
          sortable: true,
          desktopOnly: true
        }
      ]

      const data = this.filteredMarkets.map(market => ({
        id: market.id,
        slug: market.slug,
        quote_name: market.quote_token.symbol.name,
        contract: market.quote_token.contract,
        base_name: market.base_token.symbol.name,
        promoted: market.promoted,
        change_week: market.changeWeek,
        volume_week: market.volumeWeek,
        change24: market.change24,
        volume24: market.volume24,
        last_price: market.last_price
      }))

      return { header, data }
    },

    filteredMarkets() {
      if (!this.markets) return []

      let markets = []

      switch (this.markets_active_tab) {
        case 'all':
          markets = this.markets
          break

        case this.network.baseToken.symbol:
          markets = this.markets.filter(
            i => i.base_token.contract == this.network.baseToken.contract)
          break

        case 'USDT':
          markets = this.markets.filter(
            i => i.base_token.contract == 'tethertether'
          )
          break

        case 'fav':
          markets = this.markets.filter(
            i => this.favMarkets.includes(i.id)
          )
          break

        case 'Terraformers':
          markets = this.markets.filter(
            i => i.quote_token.contract == 'unboundtoken'
          )
          break

        default: {
          const ibcTokens = this.$store.state.ibcTokens.filter(
            i => i != this.network.baseToken.contract
          )
          markets = this.markets.filter((i) => {
            return (
              ibcTokens.includes(i.base_token.contract) ||
              ibcTokens.includes(i.quote_token.contract) ||
              Object.keys(this.network.withdraw).includes(i.quote_token.str) ||
              Object.keys(this.network.withdraw).includes(i.base_token.str)
            )
          })
          break
        }
      }

      markets = markets
        .filter(i => i.slug.includes(this.search.toLowerCase()) && !i.scam)
        .sort((a, b) => b.volumeWeek - a.volumeWeek)
        .reduce((res, i) => {
          i.promoted ? res.unshift(i) : res.push(i)
          return res
        }, [])

      return markets
    }
  },

  watch: {
    search() {
      this.$router.replace({ name: this.$route.name, query: { search: this.search } })
    }
  },

  mounted() {
    const { tab, search } = this.$route.query

    if (tab) {
      this.markets_active_tab = tab
    }

    if (search) {
      this.search = search
    }
  }
}
</script>

<style lang="scss">
.theme-dark {
  .markets {
    .el-input__inner {
      background-color: var(--bg-alter-2);
    }
  }
}

.theme-light .markets .el-card {
  border: none !important;
}

.markets {
  .custom-radio .el-radio-button__inner {
    padding: 8px 15px !important;
  }

}
</style>

<style lang="scss" scoped>
.table-intro {
  display: flex;
  align-items: center;
  //justify-content: space-between;
  flex-wrap: wrap;
  padding: 20px 0;

  .search-container {
    width: 450px;
  }
}

.table td,
.table th {
  border: 0 !important;
}

.last-price-item {
  width: 180px !important;
}

.pair-item {
  width: 300px !important;
}

.theme-dark {
  .markets {
    .el-card__body {
      padding: 0px;
    }
  }

  .el-input__inner {
    background-color: var(--bg-alter-2);
  }
}

.markets {
  .el-table__row {
    cursor: pointer;
  }
}

.promoted {
  float: right;
}

@media only screen and (max-width: 640px) {
  .table-intro {
    div[role="radiogroup"] {
      margin-bottom: 10px;
    }

    padding: 14px 0;

    .search-container {
      width: 70%;
    }
  }

  .el-table__row {
    .cell {
      font-size: 12px;
    }
  }

  .promoted {
    float: none;
  }
}
</style>
