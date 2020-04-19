<template>
	<div>
        <div class="window white">
            <div class='exchange'>
                <div class='exchangefields'>
                    <fieldset class='item'>
                        <legend>From:</legend>
                        <div class='maxbalance' @click='set_max_balance'>Max: <span>{{maxBalance}}</span> </div>
                        <ul>
                            <li>
                                <input type="text" id="from_currency" :disabled='disabled' name="from_currency" value='0.00'
                                :style = "{backgroundColor: fromBgColor}"
                                @input='set_to_amount'
                                v-model='fromInput'>
                                <p class='actualvalue' v-show='swapwrapped'>
                                    ≈ {{actualFromValue}} {{Object.keys(currencies)[this.from_currency] | capitalize}}
                                </p>
                            </li>
                            <li v-for='(currency, i) in Object.keys(currencies)'>
                                <input type="radio" :id="'from_cur_'+i" name="from_cur" :value='i' v-model='from_currency'>
                                <label :for="'from_cur_'+i">
                                    <span v-show='!swapwrapped'>{{currency | capitalize}}</span>
                                    <span v-show='swapwrapped'>{{currencies[currency]}}</span>
                                </label>
                            </label>
                            </li>
                        </ul>
                    </fieldset>
                    <fieldset class='item iconcontainer' @click='swapInputs'>
                        <img src='@/assets/exchange-alt-solid.svg' id='exchangeicon'/>
                    </fieldset>
                    <fieldset class='item'>
                        <legend>To:</legend>
                        <div class='maxbalance'>Max: <span></span> </div>
                        <ul>
                            <li>
                                <input type="text" 
                                id="to_currency" 
                                name="to_currency" 
                                value="0.00" 
                                disabled
                                :style = "{backgroundColor: bgColor}"
                                v-model='toInput'>
                                <p class='actualvalue' v-show='swapwrapped'>
                                    ≈ {{actualToValue}} {{Object.keys(currencies)[this.to_currency] | capitalize}}
                                </p>
                            </li>
                            <li v-for='(currency, i) in Object.keys(currencies)'>
                                <input type="radio" :id="'to_cur_'+i" name="to_cur" :value='i' v-model='to_currency'>
                                <label :for="'to_cur_'+i">
                                    <span v-show='!swapwrapped'>{{currency | capitalize}}</span>
                                    <span v-show='swapwrapped'>{{currencies[currency]}}</span>
                                </label>
                            </label>
                            </li>
                        </ul>
                    </fieldset>
                </div>
                <p class='exchange-rate'>Exchange rate (including fees): <span id="exchange-rate">{{exchangeRate}}</span></p>
                <div id='max_slippage'><span>Max slippage:</span> 
                    <input id="slippage05" type="radio" name="slippage" value='0.005' @click='maxSlippage = 0.5; customSlippageDisabled = true'>
                    <label for="slippage05">0.5%</label>

                    <input id="slippage1" type="radio" name="slippage" checked value='0.01' @click='maxSlippage = 1; customSlippageDisabled = true'>
                    <label for="slippage1">1%</label>

                    <input id="custom_slippage" type="radio" name="slippage" value='-' @click='customippageDisabled = false'>
                    <label for="custom_slippage" @click='customSlippageDisabled = false'>
                        <input type="text" id="custom_slippage_input" :disabled='customSlippageDisabled' name="custom_slippage_input" v-model='maxInputSlippage'> %
                    </label>
                </div>
                <ul>
                    <li>
                        <input id="inf-approval" type="checkbox" name="inf-approval" checked v-model='inf_approval'>
                        <label for="inf-approval">Infinite approval - trust this contract forever
                            <span class='tooltip'>[?]
                                <span class='tooltiptext long'>
                                    Preapprove the contract to to be able to spend any amount of your coins. You will not need to approve again.
                                </span>
                            </span>
                        </label>
                    </li>
                    <li v-show = "currentPool != 'susd'">
                        <input id='swapw' type='checkbox' name='swapw' v-model = 'swapwrapped'>
                        <label for='swapw'>Swap wrapped</label>
                    </li>
                </ul>
                <p class='trade-buttons'>
                    <button id="trade" @click='handle_trade'>Sell</button>
                </p>
            </div>
        </div>
</div>
</template>

<script>
    import * as common from '../utils/common.js'
    import { allCurrencies, getters, contract as state, gas as contractGas} from '../contract'
    import * as helpers from '../utils/helpers'
    import allabis from '../allabis'

    import BigNumber from 'bignumber.js'
    var cBN = (val) => new BigNumber(val);



	export default {
        data: () => ({
            disabled: true,
            from_currency: 0,
            to_currency: 1,
            inf_approval: true,
            fromInput: 0,
            toInput: 0,
            maxBalance: 0,
            promise: helpers.makeCancelable(Promise.resolve()),
            exchangeRate: 'Not available',
            bgColor: '#505070',
            fromBgColor: 'blue',
            maxSlippage: 1,
            maxInputSlippage: '',
            customSlippageDisabled: true,
            swapwrapped: false,
        }),
        created() {
            this.$watch(()=>state.default_account, (val, oldval) => {
                if(!oldval) return;
                if(val.toLowerCase() != oldval.toLowerCase()) this.mounted();
            })
            this.$watch(()=>this.initializedContracts, val => {
                if(val) this.mounted();
                console.timeEnd('initswap')
            })
        },
        watch: {
            from_currency(val, oldval) {
                if(val == this.to_currency) {
                    this.to_currency = oldval;
                }
                
                this.from_cur_handler()
            },
            to_currency(val, oldval) {
                this.to_cur_handler()
            },
            swapwrapped() {
                this.mounted()
            }
        },
        computed: {
            precisions() {
                if(this.swapwrapped) return allabis[state.currentName].wrapped_precisions;
                if(!this.isSUSD) return allabis[state.currentName].coin_precisions
                return allabis.susd.coin_precisions.concat(allabis.iearn.coin_precisions)
                if(this.from_currency < 2) return allabis.susd.coin_precisions;
                return allabis.iearn.coin_precisions
            },
            actualFromValue() {
                if(!this.swapwrapped) return;
                return (this.fromInput * this.c_rates(this.from_currency) * this.precisions[this.from_currency]).toFixed(2)
            },
            actualToValue() {
                if(!this.swapwrapped) return;
                return (this.toInput * this.c_rates(this.to_currency) * this.precisions[this.to_currency]).toFixed(2)
            },
            isSUSD() {
                return this.currentPool == 'susd'
            },
            ...getters,
            initializedContracts() {
                if(this.isSUSD) return state.contracts.iearn.initializedContracts && state.contracts.susd.initializedContracts
                else return state.currentContract.initializedContracts
            },
            currencies() {
                if(this.isSUSD) return Object.assign({},getters.currencies(), allCurrencies.iearn)
                return getters.currencies()
            },
            currentContract() {
                if(!this.isSUSD) return state.currentContract
                //sUSD->yCurve || yCurve->sUSD
                if((this.from_currency == 0 && this.to_currency == 1) || (this.from_currency == 1 && this.to_currency == 0)) return state.contracts.susd
                //sUSD->stablecoin
                return state.contracts.iearn
            },
            actualFromCurrency() {
                if(!this.isSUSD) return this.from_currency
                if(this.from_currency < 2) return this.from_currency
                return this.from_currency-2;
            },
            actualToCurrency() {
                if(!this.isSUSD) return this.to_currency
                if(this.to_currency < 2) return this.to_currency
                return this.to_currency-2; 
            },
            coins() {
                //needs to be fixed for swap wrapped in susd?
                if(!this.isSUSD) {
                    if(this.swapwrapped)
                        return this.currentContract.coins
                    return this.currentContract.underlying_coins
                }
                else {
                    return state.contracts.susd.underlying_coins.concat(state.contracts.iearn.underlying_coins)
                }
            },
            ensureAllowanceTo() {
                if(!this.isSUSD) return this.currentContract.swap._address
                if((this.from_currency == 0 && this.to_currency == 1) || (this.from_currency == 1 && this.to_currency == 0)) 
                    return state.contracts.susd.swap._address
                if(this.from_currency == 0 || this.to_currency == 0)
                    return state.contracts.susd.deposit_zap._address
                return state.contracts.iearn.swap._address;
            },
            contractNames() {
                if(!this.isSUSD) return this.currentContract.currentName
                return ['susd', 'iearn']
            },
            gasLimit() {
                if(this.isSUSD && 
                    ((this.from_currency == 0 && this.to_currency != 1) || (this.from_currency != 1 && this.to_currency == 0)))
                    //fix for swap wrapped
                    return contractGas.swap.susd.deposit_zap
                return this.swapwrapped ? contractGas.swap[this.currentContract.currentName].exchange : contractGas.swap[this.currentContract.currentName].exchange_underlying
            }
        },
        mounted() {
            if(this.currentContract.initializedContracts) this.mounted();
        },
        methods: {        
            async mounted() {
                this.disabled = false;
                this.from_cur_handler()
            },
            async from_cur_handler() {
                console.log(this.curren)
                if (cBN(await this.coins[this.from_currency].methods.allowance(state.default_account || '0x0000000000000000000000000000000000000000', allabis[state.currentName].swap_address).call()) > state.max_allowance.div(cBN(2)))
                    this.inf_approval = true;
                else
                    this.inf_approval = false;

                await this.set_from_amount(this.from_currency);
                await this.set_to_amount();
            },
            async set_from_amount(i) {
                if(!state.default_account) {
                    this.fromInput = 0;
                    this.maxBalance = 0;
                    return;
                }
                let balance = await this.coins[i].methods.balanceOf(state.default_account || '0x0000000000000000000000000000000000000000').call();
                console.log(balance, "THE BALANCE", this.coins[i]._address)
                let amount = Math.floor(
                        100 * parseFloat(balance) / this.precisions[i]
                    ) / 100
                if (this.fromInput == '' || this.val == 0) {
                    this.fromInput = amount.toFixed(2)
                }
                this.maxBalance = amount.toFixed(2);
            },
            async set_to_amount() {
                this.promise.cancel()
                let promise = this.setAmountPromise()
                try {
                    let [dy, dy_, dx_, balance] = await promise
                    this.toInput = dy;
                    this.exchangeRate = (dy_ / dx_).toFixed(4);
                    //need to fix wrapped_precisions for sUSD
                    if(this.swapwrapped) {
                        let cdy_ = (dy_ * this.c_rates(this.actualToCurrency) * allabis[state.currentName].wrapped_precisions[this.actualToCurrency])
                        let cdx_ = (dx_ * this.c_rates(this.actualFromCurrency) * allabis[state.currentName].wrapped_precisions[this.actualFromCurrency])
                        this.exchangeRate = (cdy_ / cdx_).toFixed(4)
                    }
                    if(this.exchangeRate <= 0.98) this.bgColor = 'red'
                    else this.bgColor= '#505070'
                    if(isNaN(this.exchangeRate)) this.exchangeRate = "Not available"
                    let amount = Math.floor(
                            100 * parseFloat(balance) / this.precisions[this.to_currency]
                        ) / 100

                    this.disabled = false;
                }
                catch(err) {
                    console.error(err)
                    this.disabled = true
                }
                finally {
                    this.highlight_input();
                }
                this.promise = helpers.makeCancelable(promise)
            },
            setAmountPromise() {
                let promise = new Promise(async (resolve, reject) => {
                    var i = this.from_currency;
                    var j = this.to_currency;
                    let actuali = this.actualFromCurrency;
                    let actualj = this.actualToCurrency
                    var dx_ = this.fromInput;
                    var dx = cBN(Math.round(dx_ * this.precisions[this.from_currency])).toFixed(0,1);
                    let calls = [
                        [this.currentContract.swap._address, this.currentContract.swap.methods.balances(actuali).encodeABI()],
                    ]
                    if(!this.swapwrapped)
                        calls.push(this.get_dy_underlying(dx))
                    //need to fix for susd swapwrapped
                    else {
                        //dx = cBN(dx).times(currentContract.c_rates[i])
                        calls.push([this.currentContract.swap._address, this.currentContract.swap.methods.get_dy(i, j, dx).encodeABI()])
                    }
                    calls.push([this.coins[this.to_currency]._address , this.coins[this.to_currency].methods.balanceOf(state.default_account || '0x0000000000000000000000000000000000000000').encodeABI()])
                    let aggcalls = await state.multicall.methods.aggregate(calls).call()
                    let decoded = aggcalls[1].map(hex => web3.eth.abi.decodeParameter('uint256', hex))
                    let [b, get_dy_underlying, balance] = decoded
                    b = +b * this.c_rates(i);
                    if (b >= 0.001) {
                        // In c-units
                        var dy_ = +get_dy_underlying / this.precisions[this.to_currency];
                        var dy = dy_.toFixed(2);
                        resolve([dy, dy_, dx_, balance])
                    }
                    else { 
                        this.toInput = 0
                        reject()
                    }
                })
                return helpers.makeCancelable(promise);
            },
            swapInputs() {
                //look no temp variable! :D
                [this.fromInput, this.toInput] = [this.toInput, this.fromInput]
                this.from_currency = this.to_currency
                this.from_cur_handler()
            },
            c_rates(i) {
                if(!this.isSUSD) return this.currentContract.c_rates[i]
                if(i < 2) return state.contracts.susd.c_rates[i]
                return state.contracts.iearn.c_rates[i-2];
            },
            get_dy_underlying(dx) {
                if(!this.isSUSD) 
                    return [this.currentContract.swap._address, this.currentContract.swap.methods.get_dy_underlying(this.from_currency, this.to_currency, dx).encodeABI()]
                //sUSD->yCurve || yCurve->sUSD
                if((this.from_currency == 0 && this.to_currency == 1) || (this.from_currency == 1 && this.to_currency == 0)) 
                    return [state.contracts.susd.swap._address, state.contracts.susd.swap.methods.get_dy_underlying(this.from_currency, this.to_currency, dx).encodeABI()]
                //sUSD -> stablecoin || stablecoin -> sUSD
                if(this.from_currency == 0 || this.to_currency == 0) {
                    //coins are sUSD, DAI, USDC, USDT, TUSD
                    let i = this.from_currency;
                    let j = this.to_currency;
                    if(i > 0) i = i - 1
                    if(j > 0) j = j - 1
                    return [state.contracts.susd.deposit_zap._address, state.contracts.susd.deposit_zap.methods.get_dy_underlying(i, j, dx).encodeABI()]
                }
                return [this.currentContract.swap._address, this.currentContract.swap.methods.get_dy_underlying(this.actualFromCurrency, this.actualToCurrency, dx).encodeABI()]
            },
            exchangeMethod(dx, min_dy) {
                if(!this.isSUSD) {
                    if(this.swapwrapped) return this.currentContract.swap.methods.exchange(this.from_currency, this.to_currency, dx, min_dy)
                    return this.currentContract.swap.methods.exchange_underlying(this.from_currency, this.to_currency, dx, min_dy)
                }
                if((this.from_currency == 0 && this.to_currency == 1) || (this.from_currency == 1 && this.to_currency == 0)) 
                    return state.contracts.susd.swap.methods.exchange_underlying(this.from_currency, this.to_currency, dx, min_dy)
                if(this.from_currency == 0 || this.to_currency == 0) {
                    let i = this.from_currency;
                    let j = this.to_currency
                    if(i > 0) i = i - 1
                    if(j > 0) j = j - 1
                    //for tests set min_dy to 0
                    // console.log(min_dy, "min_dy zap")
                    // min_dy = 0
                    return state.contracts.susd.deposit_zap.methods.exchange_underlying(i, j, dx, min_dy)
                }
                return state.contracts.iearn.swap.methods.exchange_underlying(this.actualFromCurrency, this.actualToCurrency, dx, min_dy)

            },
            async to_cur_handler() {
                if (this.to_currency == this.from_currency) {
                    if (this.to_currency == 0) {
                        this.from_currency = 1;
                    } else {
                        this.from_currency = 0;
                    }
                    await this.set_from_amount(this.from_currency);
                }
                await this.set_to_amount();
            },
            async set_max_balance() {
                let balance = await this.coins[this.from_currency].methods.balanceOf(state.default_account || '0x0000000000000000000000000000000000000000').call();
                let amount = Math.floor(
                        100 * parseFloat(balance) / this.precisions[this.from_currency]
                    ) / 100
                this.fromInput = amount.toFixed(2)
                await this.set_to_amount();
            },
            async highlight_input() {
                let balance = parseFloat(await this.coins[this.from_currency].methods.balanceOf(state.default_account || '0x0000000000000000000000000000000000000000').call()) /
                        this.precisions[this.from_currency];
                if (this.fromInput > balance)
                    this.fromBgColor = 'red'
                else
                    this.fromBgColor = 'blue'
            },
            async handle_trade() {
                var i = this.from_currency
                var j = this.to_currency;
                let actuali = this.actualFromCurrency
                let actualj = this.actualToCurrency
                var b = parseInt(await this.currentContract.swap.methods.balances(actuali).call()) / this.currentContract.c_rates[i];
                let maxSlippage = this.maxSlippage / 100;
                if(this.maxInputSlippage) maxSlippage = this.maxInputSlippage / 100;
                if (b >= 0.001) {
                    var dx = Math.floor(this.fromInput * this.precisions[i]);
                    var min_dy = Math.floor(this.toInput * (1-maxSlippage) * this.precisions[j]);
                    dx = cBN(dx.toString()).toFixed(0);
                    if (this.inf_approval)
                        await common.ensure_underlying_allowance(i, state.max_allowance, this.coins, this.ensureAllowanceTo, this.swapwrapped)
                    else
                        await common.ensure_underlying_allowance(i, dx, this.coins, this.ensureAllowanceTo, this.swapwrapped);
                    min_dy = cBN(min_dy.toString()).toFixed(0);
                    await this.exchangeMethod(dx, min_dy).send({
                            from: state.default_account || '0x0000000000000000000000000000000000000000',
                            gas: this.gasLimit,
                        });
                    //update both fee infos
                    await common.update_fee_infos(this.contractNames);
                    this.from_cur_handler();
                    let balance = await this.coins[i].methods.balanceOf(state.default_account || '0x0000000000000000000000000000000000000000').call();
                    let amount = Math.floor(
                            100 * parseFloat(balance) / this.precisions[i]
                        ) / 100
                    this.maxBalance = amount;
                }
            }
        }
    }
</script>

<style scoped>
	.actualvalue {
        margin: 0.5em 0 0 0;
        text-align: right;
        font-size: 0.9em;
    }
</style>