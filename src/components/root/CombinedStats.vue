<template>
	<div>
		<div class="error window half-width info" id="error-window" v-show='error'>
          {{error}}
        </div>

	    <total-balances/>

		<div class="window white" v-for='(currency, i) in Object.keys(allCurrencies)'>
		      <p class='text-center'>
		      	<router-link :to="currency" v-show="currency != 'susd'">{{currency == 'iearn' ? 'y' : currency}}.curve.fi</router-link>
		      	<a href='https://iearn.finance/pool' v-show="currency == 'susd'">susd</a>
		      </p>
		      <stats :pool= 'currency'/>
		      <balances-info 
			      :bal_info = 'currentContract.contracts[currency].bal_info'
			      :total = 'currentContract.contracts[currency].total'
			      :l_info = 'currentContract.contracts[currency].l_info'
			      :totalShare = 'currentContract.contracts[currency].totalShare'
			      :fee = 'currentContract.contracts[currency].fee * 100'
			      :admin_fee = 'currentContract.contracts[currency].admin_fee'
			      :pool = 'currency'
			      :currencies = 'allCurrencies[currency]'
			      />
	  	</div>
	</div>
</template>

<script>
	import Stats from '../Stats.vue'
	import BalancesInfo from '../BalancesInfo.vue'
	import Web3 from 'web3'
	import TotalBalances from './TotalBalances.vue'

    import { getters, contract as currentContract, allCurrencies, init } from '../../contract'
    import contracts, { infura_url, ERC20_abi, cERC20_abi, yERC20_abi } from '../../allabis'
    import * as common from '../../utils/common'

    import * as helpers from '../../utils/helpers'
    //slice(83,84) calls get_virtual_price on the token contract, not swap contract
	export default {
		metaInfo: {
	      title: 'Curve.fi - Stats',
	      meta: [
	      	{'property': 'og:url', 'content': 'https://curve.fi/combinedstats'},
	      	{'name': 'twitter:url', 'content': 'https://curve.fi/combinedstats'},
	      ]
	    },
		components: {
			Stats,
			BalancesInfo,
			TotalBalances,
		},
		data: () => ({
			contracts: [],
			web3contracts: {},
			all_coins: {},
			all_underlying_coins: {},
			all_c_rates: {},
			all_fees: {},
			totals: [],
			bal_infos: {},
			l_infos: {},
			totalShares: [],
			fees: [],
			admin_fees: [],
		}),
		created() {
            this.$watch(()=>currentContract.multicall, val => {
                this.mounted();
            })
        },
        computed: {
          currentContract() {
          	return currentContract
          },
          allCurrencies() {
          	return Object.assign(allCurrencies, {
          		susd: {
          			susd: 'sUSD',
          			ycurve: 'yCurve'
          		}
          	})
          },
          allContracts() {
          	let pools = {...contracts};
          	delete pools.y
          	return pools;
          },
          error() {
          	return currentContract.error
          },
        },
        mounted() {
            if(currentContract.multicall) this.mounted();
        },
		methods: {
			async mounted() {
				this.fees = Array.from(4).fill(0)
				this.admin_fees = Array.from(4).fill(0)
				this.admin_fees = Array.from(4).fill(0)
				this.admin_fees = Array.from(4).fill(0)
				await this.init_contracts();
				//await this.update_fee_info();
			},
			poolLink(pool) {
				if(pool == 'iearn') return 'y'
				if(pool == 'susd') return 'https://iearn.finance/pool'
				return pool
			},
			async init_contracts() {
				let calls = await Promise.all(Object.keys(this.allContracts).map(p => init(p, false, false)))
				await common.multiInitAllState(calls.flat(), Object.keys(this.allContracts))
			}
		}
	}
</script>