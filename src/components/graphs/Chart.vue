<template>
	<div class = 'tradeview window white'>
 		<select-component id='select_pool'/>
 		<highcharts :constructor-type="'stockChart'" :options="chartdata" ref='highcharts'></highcharts>
		<depth id='depth_chart' />
		<fieldset id='onesplit'>
			<legend class='text-center'>Swap using all Curve pools</legend>
			<one-split />
		</fieldset>
	</div>
</template>

<script>
	import Highcharts from 'highcharts'
	import HC_exporting from 'highcharts/modules/exporting';
	import HC_exporting_data from 'highcharts/modules/export-data';
	HC_exporting(Highcharts);
	HC_exporting_data(Highcharts)

	import {Chart} from 'highcharts-vue'
	import stockInit from 'highcharts/modules/stock'
	import Depth from './Depth.vue'
	import Select from './Select.vue'
	import EventBus from './EventBus'
	import tradeStore from './tradeStore'
	import stableswap_fns from '../../utils/stableswap_fns'
	import OneSplit from './OneSplit.vue'

	import Decimal from 'break_infinity.js'

	let BN = val => new Decimal(val)

	import * as Comlink from 'comlink'

	import Worker from 'worker-loader!./worker.js';
	const worker = new Worker();
	const calcWorker = Comlink.wrap(worker);


/*	Highcharts.seriesTypes.column.prototype.pointAttribs = (function(func) {
	    return function(point, state) {
	      var attribs = func.apply(this, arguments);
	      
	      var candleSeries = this.chart.series[0]; // Probably you'll need to change the index
	      var candlePoint = candleSeries.points.filter(function(p) { return p.index == point.index; })[0];
	      if(!candlePoint) return attribs;
	      var color = (candlePoint.open < candlePoint.close) ? '#007A00' : '#B70000'; // Replace with your colors
	      attribs.fill = state == 'hover' ? Highcharts.Color(color).brighten(0.3).get() : color;
	      
	      return attribs;
	    };
	}(Highcharts.seriesTypes.column.prototype.pointAttribs));*/

	Highcharts.setOptions({
		lang: {
			loading: '',
			rangeSelectorZoom: '',
		}
	})

	import { contract, allCurrencies, LENDING_PRECISION, PRECISION, changeContract, init } from '../../contract'
	import * as common from '@/utils/common'
	import * as helpers from '@/utils/helpers'

	import abis from '../../allabis'

	stockInit(Highcharts)

	export default {
		components: {
			highcharts: Chart,
			Depth,
			SelectComponent: Select,
			OneSplit,
		},
		data() {
			return {
				loading: true,	
					chartdata: {
						plotOptions: {
							candlestick: {
								color: '#B70000',
								upColor: '#007A00',
							},
							series: {
								point: {
									events: {
										click: (function(self) {
											return function() {
												let index = this.dataGroup ? this.dataGroup.start : this.index
												EventBus.$emit('changeTime', self.data.map(p=>p[this.index]))
											}
										})(this)
									}
								}
							}
						},
						navigator: {
							series: {
								lineColor: '#a6cdf1',
								color: '#f8fbfe'
							}
						},
						exporting: {
							buttons: {
								contextButton: {
									menuItems: ["printChart",
							                    "separator",
							                    "downloadPNG",
							                    "downloadJPEG",
							                    "downloadPDF",
							                    "downloadSVG",
							                    "separator",
							                    "downloadCSV",
							                    "downloadXLS",
							                    //"viewData",
							                    "openInCloud"]
								}
							}
						},
						rangeSelector: {
							selected: 4,
							allButtonsEnabled: true,
							buttonTheme: {
						        visibility: 'hidden',
						    },
						    labelStyle: {
						        //visibility: 'hidden'
						    },
				            inputPosition: {
				            	x: 0,
								align:'left',
					        },
						 	buttons: [{
		                        type: 'minute',
		                        count: 100,
		                        text: '1m',
		                        dataGrouping: {
				                    forced: true,
				                    units: [['minute', [1]]]
				                }
		                    }, 
		                    {
		                        type: 'minute',
		                        count: 500,
		                        text: '5m',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['minute', [5]]]
				                }
		                    },
		                    {
		                    	type: 'minute',
		                    	count: 1500,
		                    	text: '15m',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['minute', [15]]]
				                }
		                    },
		                    {
		                        type: 'minute',
		                        count: 3000,
		                        text: '30m',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['minute', [30]]]
				                }
		                    }, 
		                    {
		                        type: 'hour',
		                        count: 500,
		                        text: '1h',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['hour', [1]]]
				                }
		                    }, 
		                    {
		                        type: 'hour',
		                        count: 1000,
		                        text: '2h',
		                        dataGrouping: {
				                    forced: true,
				                    units: [['hour', [2]]]
				                }
		                    },
		                    {
		                        type: 'hour',
		                        count: 2000,
		                        text: '4h',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['hour', [4]]]
				                }
		                    },
		                    {
		                        type: 'hour',
		                        count: 3000,
		                        text: '6h',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['hour', [6]]]
				                }
		                    },
		                    {
		                        type: 'day',
		                        count: 30,
		                        text: '1d',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['day', [1]]]
				                }
		                    },
		                    {
		                        type: 'day',
		                        count: 90,
		                        text: '3d',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['day', [3]]]
				                }
		                    },
		                    {
		                    	type: 'week',
		                    	count: 10,
		                    	text: '1w',
		                        dataGrouping: {
			                 	   forced: true,
				                    units: [['week', [1]]]
				                }
		                    }
		                    ]
						},
						chart: {
							height: 600,
							panning: true,
							zoomType: 'x',
					        panKey: 'ctrl',
					        events: {
					        	click: (function(self) {
					        		return function(e) {
					        			let nearest = self.chart.pointer.findNearestKDPoint(self.chart.series, false, e)
					        			//console.log(nearest)
					        			let index = nearest.dataGroup ? nearest.dataGroup.start : nearest.index
					        			//console.log(self.data[nearest.sindex])
										EventBus.$emit('changeTime', self.data.map(p=>p[index]))
										//console.log(+get_dy_underlying, "price at point", index)
										//console.log(self.data[index].prices["0-1"])
					        		}
					        	})(this),
					        	load() {
					        		this.redraw();
					        	},
					        	render() {
					        		if(this.plotWidth < 480) {
					        			this.rangeSelector.maxLabel.translate(0, 25)
					        			this.rangeSelector.maxDateBox.translate(37, 25)
					        		}
					        		else {
					        			this.rangeSelector.maxLabel.translate(141, 0)
					        			this.rangeSelector.maxDateBox.translate(164, 0)
					        		}
					        	}
					        }
						},
						title: {
							text: '',
						},
						yAxis: [{
								labels: {
				                align: 'right',
					                x: -3
					            },
					            title: {
					                text: 'OHLC'
					            },
					            height: '60%',
					            lineWidth: 2,
					            resize: {
					                enabled: true
					            }
					        }, 
					        {
					            labels: {
					                align: 'right',
					                x: -3
					            },
					            title: {
					                text: 'Volume'
					            },
					            top: '65%',
					            height: '35%',
					            offset: 0,
					            lineWidth: 2
					        }],
					  	tooltip: {
					  		split: true,
					  		borderColor: '#a6cdf1',
					  	},
					  	series: [
						  	{
						  		type: 'candlestick',
					            name: this.pairVal,
					            data: [],
						  	},
						  	{
						  		type: 'column',
					            name: 'Volume',
					            data: [],
					            yAxis: 1,
						  	}
					  	],
			    	},
				  	pair: {
				  		
				  	},
				  	pool: '',
				  	pools: [],
				  	interval: null,
				  	chart: null,
				  	data: [],
				  	poolConfigs: null,
				  	fromCurrency: null,
				  	toCurrency: null,
		  }
		},
		created() {
			//EventBus.$on('selected', this.selectPool);
		},
		watch: {
			selectChange() {
				this.mounted()
			}
		},
		async mounted() {
			this.chart = this.$refs.highcharts.chart;
/*			this.$watch(()=>contract.initializedContracts, val => {
                if(val) this.mounted();
            })*/
			let pools = tradeStore.pools.map(p=>p == 'y' ? 'iearn' : p)
			let calls = await Promise.all(pools.map(p=>init(p, false, false)))
			await common.multiInitAllState(calls.flat(), pools)
            this.mounted()
		},
		beforeDestroy() {
			EventBus.$off('selected', this.selectPool)
		},
		computed: {
			selectChange() {
				return tradeStore.pairIdx, tradeStore.pools.join(), tradeStore.interval, Date.now();
			}
		},
		methods: {
			async mounted() {

				this.chart.showLoading();
				this.pools = tradeStore.pools;
				this.pairIdx = tradeStore.pairIdx
				this.pairVal = tradeStore.pairVal
				this.interval = tradeStore.interval;
/*				while(this.chart.series.length) {
					this.chart.series[0].remove()
				}*/

				//return;
				//move this to selectPool method
				let jsonInterval = this.interval;
				if(tradeStore.intervals.indexOf(jsonInterval) > 3) jsonInterval = '30m'
				let urls = tradeStore.pools.map(pool=>fetch(`https://beta.curve.fi/raw-stats/${pool == 'iearn' ? 'y' : pool}-${jsonInterval}.json`));
				let requests = await Promise.all(urls)
				let data = this.data = await Promise.all(requests.map(r=>r.json()))
				//tradeStore.data = data;
				let pools = tradeStore.pools.map(p=>p == 'y' ? 'iearn' : p)
				let poolConfigs = this.poolConfigs = pools.map(pool => {
					return {
						N_COINS: abis[pool].N_COINS,
						PRECISION_MUL: abis[pool].coin_precisions.map(p=>1e18/p),
						USE_LENDING: abis[pool].USE_LENDING,
						LENDING_PRECISION,
						PRECISION,
					}
				})

				let fromCurrency = this.fromCurrency = this.pairIdx.split('-')[0]
				let toCurrency = this.toCurrency = this.pairIdx.split('-')[1]
				let inverse = false;
				if(fromCurrency > toCurrency) {
					inverse = true;
					[fromCurrency, toCurrency] = [toCurrency, fromCurrency]
					this.pairIdx = `${fromCurrency}-${toCurrency}`
				}


			    let ohlc = []
			    let volume = []
				let ohlcData = []
				let lastPriceCalls = pools.map(name=>[contract.contracts[name].swap._address, contract.contracts[name].swap.methods.get_dy_underlying(fromCurrency, toCurrency, BN(abis[name].coin_precisions[fromCurrency]).toFixed(0)).encodeABI()])
				let aggcalls = await contract.multicall.methods.aggregate(lastPriceCalls).call()
				let lastPrices = aggcalls[1].map(hex => web3.eth.abi.decodeParameter('uint256', hex))
					//
				let length = data[0].length;
				for(let l = length-1; l > 0; l-= 100) {
					for(let i = l; i > l-100; i--) {
						ohlcData[i] = {}
						ohlcData[i].timestamp = data[0][i].timestamp
						ohlcData[i].prices = {}
						ohlcData[i].prices[this.pairIdx] = []
						ohlcData[i].volume = {}
						ohlcData[i].volume[this.pairIdx] = []
						for(let j = 0; j < data.length; j++) {
							if(poolConfigs[j].N_COINS-1 < toCurrency) continue;
							let v = data[j][i]
							//console.log(v, poolConfigs[j], poolConfigs, i, j, fromCurrency, toCurrency, "CALC CONFIG")
							let get_dy_underlying = await calcWorker.calcPrice(
								{...v, ...poolConfigs[j]}, fromCurrency, toCurrency, abis[pools[j]].coin_precisions[fromCurrency])
							let calcprice = +(BN(get_dy_underlying).div(abis[pools[j]].coin_precisions[toCurrency]))
							if(inverse) calcprice = 1 / calcprice
							if(v.prices[this.pairIdx]) {
								if(inverse) v.prices[this.pairIdx] = v.prices[this.pairIdx].map(price => 1/price)
								v.prices[this.pairIdx].push(calcprice)
							}
							else {
								v.prices[this.pairIdx] = [calcprice]
								v.volume[this.pairIdx] = [0]
							}
							if(i == length-1) {
								let dx = BN(abis[pools[j]].coin_precisions[fromCurrency]).toFixed(0)
								let lastPrice = +(BN(lastPrices[j])).div(abis[pools[j]].coin_precisions[toCurrency])
								ohlcData[i].prices[this.pairIdx].push(lastPrice)
							}
							ohlcData[i].prices[this.pairIdx].push(...v.prices[this.pairIdx])
							ohlcData[i].volume[this.pairIdx][j] = v.volume[this.pairIdx].map((v,k)=>v / abis[pools[j]].coin_precisions[k])
						}
					}

			    	// split the data set into ohlc and volume

				    let dataLength = ohlcData.length
				        // set the allowed units for data grouping

				    for (let i = l; i > l-100; i--) {
				    	let len = ohlcData[i].prices[this.pairIdx].length
	/*			    	console.log(ohlcData[i].timestamp*1000, // the date
				            ohlcData[i].prices[this.pairIdx][0], // open
				            Math.max(...ohlcData[i].prices[this.pairIdx]), // high
				            Math.min(...ohlcData[i].prices[this.pairIdx]), // low
				            ohlcData[i].prices[this.pairIdx][len-1] // close
				        )*/
				        ohlc.unshift([
				            ohlcData[i].timestamp*1000, // the date
				            ohlcData[i].prices[this.pairIdx][0], // open
				            Math.max(...ohlcData[i].prices[this.pairIdx]), // high
				            Math.min(...ohlcData[i].prices[this.pairIdx]), // low
				            ohlcData[i].prices[this.pairIdx][len-1] // close
				        ]);
				        let volumeData = ohlcData[i].volume[this.pairIdx].map(vs=>vs[0])
				        if(inverse) volumeData = ohlcData[i].volume[this.pairIdx].map(vs=>vs[1])
				        volume.unshift([
				            ohlcData[i].timestamp*1000, // the date
				            volumeData.reduce((a, b) => a + b, 0) // the volume
				        ]);
	/*
				        if(inverse) {
				        	ohlc = ohlc.map(p => p.map((v, i) => i > 0 ? 1/v : v))
				        	volume = volume.map((p, i) => {
				        		p[1] = data[i].volume[this.pairIdx][1] / abis[this.pool].coin_precisions[toCurrency]
				        		return p;
				        	})
				        }*/
				        /*if(i == dataLength/20 || i == dataLength/10 || i == dataLength/5 || i == dataLength/2 || i == dataLength-1) {
				        }*/
        		        this.chart.hideLoading();

				    }
				    this.$refs.highcharts.chart.series[0].setData(ohlc)
				    this.$refs.highcharts.chart.series[1].setData(volume)

				}


		        console.log(this.$refs.highcharts.chart)
		        this.chart.setTitle({text: this.pairVal.toUpperCase()})
/*		        this.chart.update({
		        	rangeSelector: {
		        		buttons: this.chartdata.rangeSelector.buttons.map(b=> {
		        			let cb = {...b}
		        			cb.count /= jsonInterval.slice(0,-1) / 2
		        			return cb;
		        		})
		        	}
		        })*/
		        //highcharts doesn't select the defined range, doing it again manually
		        this.chart.rangeSelector.clickButton(tradeStore.intervals.indexOf(this.interval), true)
		        this.chart.redraw();
		        this.chart.hideLoading();
			    this.loading = false;
			}
		}
	}
</script>

<style scoped>
	.tradeview {
		width: 70%;
		max-width: 1280px;
	}
	#select_pool {
		margin-bottom: 10px;
	}
	#onesplit {
		margin-top: 30px;
	}
</style>