<template lang="pug">
  div(v-if="isDrizzleInitialized")
    p
      h2 Yearn veCRV "Backscratcher" Vault
      p
        div.green Earn <strong>{{ vault_boost - 1 | toPct(1) }}</strong> more weekly rewards staking with Yearn.
      p This vault accepts CRV in exchange for perpetual claim on Curve DAO admin fees across all Yearn products.
      p Since it locks CRV in Curve Voting Escrow for 4 years and regularly prolongs the lock, this vault doesn't have withdrawal functionality.
      p
        div.green
          div 🍣 Stake <a href="https://analytics.sushi.com/pairs/0x10b47177e92ef9d5c6059055d92ddf6290848991">Sushiswap LP</a> on <a href="https://app.sushi.com/farm">Onsen menu</a> for more rewards.
          div 🥒 Stake LP in <a href="https://app.pickle.finance/jars">Pickle Jar</a> for Sushi + Pickle rewards.
    p
      div 🦍 user
      div wallet balance: {{ crv_balance | fromWei(2) }} CRV
      div vested balance: {{ vested_balance | fromWei(2) }} CRV
      div gauges claimable: {{ gauge_balance | fromWei(2) }} CRV
      div vault balance: {{ vault_balance | fromWei(2) }} yveCRV
      div claimable rewards: {{ claimable | fromWei(2) }} 3Crv
    p
      div 🧮 vault
      div curve apy: {{ vault_apy | toPct(2) }}
      div vault apy: {{ vault_apy * vault_boost | toPct(2) }}
      div vault vs solo: {{ vault_boost.toFixed(2) }}x
      div rewards in vault: {{ vault_rewards | fromWei(2) }} 3Crv
      div epochs claimed: {{ epochs_claimed }}/{{ epochs_total }}
      div total supply: {{ vault_supply | fromWei(2) }} yveCRV ({{ vault_supply / total_vecrv | toPct(3) }} of total)
      div yearn vecrv: {{ yearn_vecrv | fromWei(2) }} veCRV ({{ yearn_vecrv / total_vecrv | toPct(3) }} of total)
      div total vecrv: {{ total_vecrv | fromWei(2) }} veCRV
    p
      div 🕹 interact
      div account: {{ username || activeAccount }}
      p.muted deposit CRV from wallet into yveCRV vault
      button(:disabled='has_allowance_vault', @click.prevent='on_approve_vault') {{ has_allowance_vault ? 'vault approved' : 'approve vault' }}
      button(:disabled='!has_allowance_vault', @click.prevent='on_deposit') deposit {{ crv_balance | fromWei(2) }} CRV
      button(@click.prevent='on_claim') claim {{ claimable | fromWei(2) }} 3Crv
    p.row
      p.muted deposit CRV from wallet, vesting and gauges into yveCRV vault
      button(:disabled='has_allowance_zap', @click.prevent='on_approve_zap') {{ has_allowance_zap ? 'CRV zap approved' : 'approve CRV zap' }}
      button(:disabled='minting_allowed', @click.prevent='on_approve_minter') {{ minting_allowed ? 'minter approved' : 'approve minter' }}
      button(:disabled='!has_allowance_zap || (need_minter && !minting_allowed)', @click.prevent='on_zap') zap {{ zap_balance | fromWei(2) }} CRV
    p.row
      div.muted
        div vault by 
          a(href='https://twitter.com/andrecronjetech', target='_blank') andre
          span , ui by 
          a(href='https://twitter.com/bantg', target='_blank') banteg
          span , zaps by banteg and kx9x
        div
          a(:href='`https://etherscan.io/address/${vault}/#code`', target='_blank') vault contract
          span , 
          a(:href='"https://etherscan.io/address/" + zap + "/#code"', target='_blank') zap deposit contract
          span , 
          a(:href='"https://etherscan.io/address/" + y3crv_zap + "/#code"', target='_blank') zap rewards contract
        div
          a(href='https://github.com/banteg/ape-tax', target='_blank') source code
          span , 
          a(href='#', @click.prevent='on_add_token') add to metamask
          span

</template>

<script>
import { mapGetters } from 'vuex'
import ethers from 'ethers'
import CurveGauge from './abi/CurveGauge.json'
const max_uint = new ethers.BigNumber.from(2).pow(256).sub(1).toString()
const ZERO_ADDRESS = '0x0000000000000000000000000000000000000000'

export default {
  name: 'veCurveVault',
  data() {
    return {
      user_gauges: [],
      gauge_balance: 0,
      username: null,
      crv_price: 0,
      yvecrv_price: 0,
    }
  },
  filters: {
    fromWei(data, precision) {
      if (data === 'loading') return data
      if (data > 2**255) return '∞'
      let value = ethers.utils.commify(ethers.utils.formatEther(data))
      let parts = value.split('.')
      return parts[0] + '.' + parts[1].slice(0, precision)
    },
    toPct(data, precision) {
      if (isNaN(data)) return '?'
      return `${(data * 100).toFixed(precision)}%`
    }
  },
  methods: {
    on_approve_vault() {
      this.drizzleInstance.contracts['CRV'].methods['approve'].cacheSend(this.vault, max_uint, {from: this.activeAccount})
    },
    on_approve_zap() {
      this.drizzleInstance.contracts['CRV'].methods['approve'].cacheSend(this.zap, max_uint, {from: this.activeAccount})
    },
    on_approve_minter() {
      this.drizzleInstance.contracts['CurveMinter'].methods['toggle_approve_mint'].cacheSend(this.zap, {from: this.activeAccount})
    },
    on_deposit() {
      this.drizzleInstance.contracts['veCurveVault'].methods['depositAll'].cacheSend({from: this.activeAccount})
    },
    on_zap() {
      this.drizzleInstance.contracts['CurveBackzapper'].methods['zap'].cacheSend(this.user_gauges, {from: this.activeAccount})
    },
    on_approve_y3crv_zap() {
      this.drizzleInstance.contracts['3CRV'].methods['approve'].cacheSend(this.y3crv_zap, max_uint, {from: this.activeAccount})
    },
    on_y3crv_zap() {
      this.drizzleInstance.contracts['y3CrvZapper'].methods['zap'].cacheSend({from: this.activeAccount})
    },
    on_claim() {
      this.drizzleInstance.contracts['veCurveVault'].methods['claim'].cacheSend({from: this.activeAccount})
    },
    on_add_token() {
      window.ethereum.request({
        method: 'wallet_watchAsset',
            params: {
              'type': 'ERC20',
              'options': {
                'address': this.vault,
                'symbol': 'yveCRV',
                'decimals': 18,
              },
            }
        })
    },

    async load_user_gauges() {
      let pool_count = await this.drizzleInstance.contracts['CurveRegistry'].methods['pool_count']().call()
      let pools = await Promise.all([...Array(parseInt(pool_count)).keys()].map(
        i => this.drizzleInstance.contracts['CurveRegistry'].methods['pool_list'](i).call()
      ))
      let resp = await Promise.all(pools.map(
        pool => this.drizzleInstance.contracts['CurveRegistry'].methods['get_gauges'](pool).call()
      ))
      let gauges = []
      for (let x of resp) gauges.push(...x[0])
      gauges = gauges.filter(gauge => gauge !== ZERO_ADDRESS)
      let balances = await Promise.all(gauges.map(this.gague_claimable))
      let user_gauges = gauges.filter((gauge, i) => balances[i].gt(0))
      while (user_gauges.length < 20) user_gauges.push(ZERO_ADDRESS)
      this.gauge_balance = balances.reduce((prev, curr) => curr.add(prev))
      this.user_gauges = user_gauges
    },

    async load_reverse_ens() {
      let lookup = this.activeAccount.toLowerCase().substr(2) + '.addr.reverse'
      let resolver = await this.drizzleInstance.web3.eth.ens.resolver(lookup)
      let namehash = ethers.utils.namehash(lookup)
      this.username = await resolver.methods.name(namehash).call()
    },

    async gague_claimable(gauge) {
      let contract = new this.drizzleInstance.web3.eth.Contract(CurveGauge, gauge)
      let balance = await contract.methods.claimable_tokens(this.activeAccount).call()
      return new ethers.BigNumber.from(balance)
    },

    async load_crv_price() {
      const resp = await fetch('https://api.coingecko.com/api/v3/simple/price?ids=curve-dao-token,vecrv-dao-yvault&vs_currencies=usd')
      const data = await resp.json()
      this.crv_price = data['curve-dao-token']['usd']
      this.yvecrv_price = data['vecrv-dao-yvault']['usd']
    },

    call(contract, method, args, out='number') {
      let key = this.drizzleInstance.contracts[contract].methods[method].cacheCall(...args)
      let value
      try {
        value = this.contractInstances[contract][method][key].value 
      } catch (error) {
        value = null
      }
      switch (out) {
        case 'number':
          if (value === null) value = 0
          return new ethers.BigNumber.from(value)
        case 'address':
          return value
        default:
          return value
      }
    },
  },
  computed: {
    ...mapGetters('accounts', ['activeAccount', 'activeBalance']),
    ...mapGetters('drizzle', ['drizzleInstance', 'isDrizzleInitialized']),
    ...mapGetters('contracts', ['getContractData', 'contractInstances']),

    user() {
      return this.activeAccount
    },
    vault() {
      return this.drizzleInstance.contracts['veCurveVault'].address
    },
    crv() {
      return this.drizzleInstance.contracts['CRV'].address
    },
    zap() {
      return this.drizzleInstance.contracts['CurveBackzapper'].address
    },
    y3crv_zap() {
      return this.drizzleInstance.contracts['y3CrvZapper'].address
    },
    voter() {
      return this.drizzleInstance.contracts['CurveYCRVVoter'].address
    },
    vault_supply() {
      return this.call('veCurveVault', 'totalSupply', [])
    },
    crv_balance() {
      return this.call('CRV', 'balanceOf', [this.activeAccount])
    },
    three_crv_balance() {
      return this.call('3CRV', 'balanceOf', [this.activeAccount])
    },
    vested_balance() {
      return this.call('CurveVesting', 'balanceOf', [this.activeAccount])
    },
    zap_balance() {
      return this.crv_balance.add(this.vested_balance).add(this.gauge_balance)
    },
    crv_allowance() {
      return this.call('CRV', 'allowance', [this.activeAccount, this.vault])
    },
    vault_balance() {
      return this.call('veCurveVault', 'balanceOf', [this.activeAccount])
    },
    claimable() {
      const cached = this.call('veCurveVault', 'claimable', [this.activeAccount])
      const index = this.call('veCurveVault', 'index', [])
      const supply_index = this.call('veCurveVault', 'supplyIndex', [this.activeAccount])
      const ratio = new ethers.BigNumber.from(10).pow(18)
      return (index.sub(supply_index)).mul(this.vault_balance).div(ratio).add(cached)
    },
    vault_apy() {
      const week = 7 * 86400
      const epoch = Math.floor(Date.now() / 1000 / week) * week - week
      const tokens_per_week = this.call('CurveRewardDistribution', 'tokens_per_week', [epoch]) / 1e18
      const virtual_price = this.call('Curve3pool', 'get_virtual_price', []) / 1e18
      return (tokens_per_week * virtual_price * 52) / (this.total_vecrv / 1e18 * this.crv_price)
    },
    vault_boost()  {
      return (this.yearn_vecrv / this.vault_supply) * (this.crv_price / this.yvecrv_price)
    },
    three_crv_zappable() {
      return this.three_crv_balance.add(this.claimable)
    },
    yearn_vecrv() {
      return this.call('CurveVotingEscrow', 'balanceOf', [this.voter])
    },
    total_vecrv() {
      return this.call('CurveVotingEscrow', 'totalSupply', [])
    },
    vault_rewards() {
      return this.call('3CRV', 'balanceOf', [this.vault])
    },
    epochs_claimed() {
      return this.call('CurveRewardDistribution', 'user_epoch_of', [this.voter])
    },
    epochs_total() {
      return this.call('CurveVotingEscrow', 'user_point_epoch', [this.voter])
    },
    has_allowance_vault() {
      return !this.call('CRV', 'allowance', [this.activeAccount, this.vault]).isZero()
    },
    has_allowance_zap() {
      return !this.call('CRV', 'allowance', [this.activeAccount, this.zap]).isZero()
    },
    has_allowance_y3crv_zap() {
      return !this.call('3CRV', 'allowance', [this.activeAccount, this.y3crv_zap]).isZero()
    },
    need_minter() {
      return this.user_gauges.filter(gauge => gauge !== ZERO_ADDRESS).length > 0
    },
    minting_allowed() {
      return this.call('CurveMinter', 'allowed_to_mint_for', [this.zap, this.activeAccount], 'object')
    },
  },
  created() {
    this.load_user_gauges()
    this.load_reverse_ens()
    this.load_crv_price()
  }
}
</script>

<style>
button {
  margin-right: 1em;
}
.muted {
  color: gray;
  font-size: 0.8em;
}
.green {
  background-color: #94FF63;
  box-shadow: 0 0 8px 6px #94FF63;
  display: inline-block;
}
a, a:visited, a:hover {
  color: black;
}
</style>
