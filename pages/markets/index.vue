<template lang="pug">
.markets
  market-top(v-if="!isMobile" :newListings="newListings", :topGainers="topGainers", :topVolume="topVolume")
  .table-intro
    el-radio-group.radio-chain-select.custom-radio(
      v-model='markets_active_tab',
      size='small'
    ).mr-3
      el-radio-button(:label='$t("fav")')
        i.el-icon-star-on
        span {{ $t('Fav') }}

      el-radio-button(:label='$t("all")')
        span {{ $t('All') }}

      el-radio-button(:label='network.baseToken.symbol')
        span {{ network.baseToken.symbol }}

      el-radio-button(v-if='network.name == "eos"', label='USDT')
        span {{ $t('USDT') }}

      el-radio-button(value='cross-chain', :label='$t("Cross-Chain")')
        span {{ $t('Cross-Chain') }}

    .search-container
      el-input(
        v-model='search',
        :placeholder='$t("Search market")',
        size='small',
        prefix-icon='el-icon-search'
        clearable
      )

    el-switch(v-if="markets_active_tab == network.baseToken.symbol" v-model='showVolumeInUSD' active-text='USD').ml-auto

    .ml-auto(v-if="!isMobile")
      nuxt-link(:to="localePath('new_market', $i18n.locale)")
        el-button(tag="el-button" size="small" icon="el-icon-circle-plus-outline") {{ $t('Open new market') }}

  virtual-table(:table="virtualTableData")
    template(#row="{ item }")
      market-row(:item="item" :showVolumeInUSD="showVolumeInUSD" :marketsActiveTab="markets_active_tab")
</template>

<script>
import { mapState } from 'vuex'
import VirtualTable from '~/components/VirtualTable'
import MarketRow from '~/components/MarketRow'
import MarketTop from '~/components/MarketTop'

export default {
  components: {
    VirtualTable,
    MarketRow,
    MarketTop
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
      search: '',
      newListingIDs: [495, 455, 173],
      topGaindersIDs: [475, 422, 416]
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

    newListings() {
      return this.markets.filter(({ id }) => this.newListingIDs.includes(id))
    },

    topGainers() {
      const tmp = [...this.markets]
      return tmp
        .filter(i => i.base_token.contract == this.network.baseToken.contract)
        .sort((a, b) => b.volumeWeek - a.volumeWeek)
        .slice(0, 50)
        .sort((a, b) => b.change24 - a.change24)
        .slice(0, 3)
    },

    topVolume() {
      const tmp = [...this.markets]
      return tmp
        .filter(i => i.base_token.contract == this.network.baseToken.contract)
        .sort((a, b) => b.volume24 - a.volume24)
        .slice(0, 3)
    },

    virtualTableData() {
      const header = [
        {
          label: 'Pair',
          value: 'quote_name',
          width: '335px',
          sortable: true
        },
        {
          label: 'Last price',
          value: 'last_price',
          width: '165px',
          sortable: true
        },
        {
          label: '24H Vol.',
          value: 'volume24',
          width: '190px',
          sortable: true,
          desktopOnly: true
        },
        {
          label: '24H',
          value: 'change24',
          width: '100px',
          sortable: true,
          desktopOnly: true
        },
        {
          label: '7D Volume',
          value: 'volume_week',
          width: '190px',
          sortable: true
        },
        {
          label: '7D Change',
          value: 'change_week',
          width: '100px',
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

      const itemSize = 55
      const pageMode = true

      return { pageMode, itemSize, header, data }
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
          i.promoted ? res[0].push(i) : res[1].push(i)
          return res
        }, [[], []])
        .reduce((res, subArr) => {
          res.push(...subArr)
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

.red {
  color: var(--main-red);
}

.green {
  color: var(--main-green);
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
