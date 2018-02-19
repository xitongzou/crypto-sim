<template>
  <div id="app">
    <img src="./assets/logo.png">
    <header>
      <h1>{{ msg }}</h1>
    </header>
    <div class="main">
      <form class="inputForm">
        <md-field>
          <label>Coin Symbol (currently only supports BTC):</label>
          <md-input v-model="coin"></md-input>
        </md-field>
        <md-field>
          <label>Start Date: </label>
          <md-input v-model="startDate"></md-input>
        </md-field>
        <md-field>
          <label>End Date: </label>
          <md-input v-model="endDate"></md-input>
        </md-field>
        <md-field>
          <label>Cash on hand: </label>
          <md-input v-model="buyingPower" type="number"></md-input>
        </md-field>
        <md-field>
          <label>Commission fees as a percent (set it to whatever your exchange charges): </label>
          <md-input v-model="fees" type="number"></md-input>
        </md-field>
        <md-button v-on:click="simulate">Simulate</md-button>
      </form>
      <div id="chartContainer">
        <img className="spinner" role="presentation" src="/spinner.gif" v-if="spinnerVisible" />
      </div>
      <div id="simulationContainer">
        <img className="spinner" role="presentation" src="/spinner.gif" v-if="spinnerVisible" />
      </div>
      <div v-html="analysis">
        <h2 className="logTitle">Full Log:</h2>
        {{ analysis }}
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import highcharts from 'highcharts';

export default {
  name: 'App',
  data: function () {
    return {
      bpi: [],
      msg: 'Welcome to Tongs Crypto Sim App',
      coin: 'BTC',
      startDate: '2011-01-01',
      endDate: '2018-01-01',
      buyingPower: '10000',
      fees: '0.25',
      analysis: '',
      spinnerVisible: false,
      simulationPoints: []
    }
  },
  methods: {
   addSimulationPoint (name, date, action, numShares, price) {
       this.simulationPoints = this.simulationPoints.concat({name, date, action, numShares, price})
   },

   renderChart (coin, dates, prices) {
        highcharts.chart('chartContainer', {
            chart: {
                zoomType: 'x'
            },
            title: {
                text: 'Simulation for ' + coin + ' from ' + this.startDate + ' to ' + this.endDate,
                x: -20 //center
            },
            subtitle: {
                text: 'Source: api.oindesk.com',
                x: -20
            },
            xAxis: {
                categories: dates
            },
            yAxis: {
                title: {
                    text: 'Price (USD)'
                },
                plotLines: [{
                    value: 0,
                    width: 1,
                    color: '#808080'
                }]
            },
            tooltip: {
                valuePrefix: '$'
            },
            legend: {
                layout: 'vertical',
                align: 'right',
                verticalAlign: 'middle',
                borderWidth: 0
            },
            plotOptions: {
                area: {
                    fillColor: {
                        linearGradient: {
                            x1: 0,
                            y1: 0,
                            x2: 0,
                            y2: 1
                        },
                        stops: [
                            [0, highcharts.getOptions().colors[0]],
                            [1, highcharts.Color(highcharts.getOptions().colors[0]).setOpacity(0).get('rgba')]
                        ]
                    },
                    marker: {
                        radius: 2
                    },
                    lineWidth: 1,
                    states: {
                        hover: {
                            lineWidth: 1
                        }
                    },
                    threshold: null
                }
            },
            series: [{
                type: 'area',
                name: coin,
                data: prices
            }]
        });
    },

   renderSimulation(coin, dates, prices) {
        // stupid; highslideJS doesn't have an npm package so have to reference it client-side
        const hs = window.hs;
        const simulationPoints = this.simulationPoints
        highcharts.chart('simulationContainer', {

            title: {
                text: 'Simulation data points for ' + coin
            },

            subtitle: {
                text: 'Click on each data point for information'
            },

            xAxis: {
                categories: dates
            },

            yAxis: [{ // left y axis
                title: {
                    text: 'Price (USD)'
                },
                labels: {
                    align: 'left',
                    x: 3,
                    y: 16,
                    format: '{value:.,0f}'
                },
                showFirstLabel: false
            }, { // right y axis
                linkedTo: 0,
                gridLineWidth: 0,
                opposite: true,
                title: {
                    text: null
                },
                labels: {
                    align: 'right',
                    x: -3,
                    y: 16,
                    format: '{value:.,0f}'
                },
                showFirstLabel: false
            }],

            legend: {
                align: 'left',
                verticalAlign: 'top',
                y: 20,
                floating: true,
                borderWidth: 0
            },

            tooltip: {
                shared: true,
                crosshairs: true
            },

            plotOptions: {
                series: {
                    cursor: 'pointer',
                    point: {
                        events: {
                            click: function (e) {
                                hs.htmlExpand(null, {
                                    pageOrigin: {
                                        x: e.pageX || e.clientX,
                                        y: e.pageY || e.clientY
                                    },
                                    headingText: this.category,
                                    maincontentText: simulationPoints[this.index].action + ' ' +
                                    simulationPoints[this.index].numCoins + ' ' + coin + ' at $' + this.y,
                                    width: 200
                                });
                            }
                        }
                    },
                    marker: {
                        lineWidth: 1
                    }
                }
            },

            series: [{
                name: coin,
                data: prices
            }]
        });
    },

    simulate (e) {
      this.spinnerVisible = true;
      const that = this
      axios.get('https://api.coindesk.com/v1/bpi/historical/close.jsonp?start=' + this.startDate + '&end=' + this.endDate)
          .then(function (response) {

            that.bpi = response.data.bpi
            const dates = Object.keys(that.bpi)
            const prices = Object.values(that.bpi)
            that.spinnerVisible = true;

            that.analysis = '<h3>Analyzing predictions for BTC...</h3>'

            if (dates.length > 0 && prices.length > 0) {

                let ema12 = 0.0
                let ema26 = 0.0
                let macd = []
                let signal = []
                let masum12 = 0.0
                let masum26 = 0.0
                let mamacd = 0.0
                let buyPrice = 0.0
                let nextPrice = 0.0
                let profit = 0.0
                let numCoins = 0
                let bought = false
                let sold = false
                let commission = 0
                let success = 0
                let trades = 0
                let support1 = 0.0
                let support2 = 0.0
                let support3 = 0.0
                let resistance1 = 0.0
                let resistance2 = 0.0
                let resistance3 = 0.0

                // Calculate 12-day and 26-day moving average (EMA) and 9-day MACD
                // fibonacci extensions are 0.382, 0.5 and 0.618 for support, and 1.382, 1.5 and 1.618 for resistance
                for (let i = 0; i < 35; i++) {
                    let price = prices[i]
                    masum26 += price
                    masum12 += price

                    if (i >= 12) {
                       masum12 -= prices[i-12]
                       ema12 = masum12 / 12
                    }

                    if (i >= 26) {
                       masum26 -= prices[i-26]
                       ema26 = masum26 / 26
                       macd[i] = ema12 - ema26
                       mamacd += macd[i]
                    }
                }

                signal[34] = mamacd / 9

                support1 = prices[34] * 0.618
                support2 = prices[34] * 0.5
                support3 = prices[34] * 0.382
                resistance1 = prices[34] * 1.382
                resistance2 = prices[34] * 1.5
                resistance3 = prices[34] * 1.618

                // main loop
                for (let i = 35; i < dates.length; i++) {
                  let price = prices[i]
                  nextPrice = price
                  masum26 += price
                  masum12 += price
                  masum26 -= prices[i-26]
                  masum12 -= prices[i-12]
                  ema26 = masum26 / 26
                  ema12 = masum12 / 12
                  macd[i] = ema12 - ema26
                  mamacd += macd[i]
                  mamacd -= macd[i-9]
                  signal[i] = mamacd / 9
                  support1 = price * 0.618
                  support2 = price * 0.5
                  support3 = price * 0.382
                  resistance1 = price * 1.382
                  resistance2 = price * 1.5
                  resistance3 = price * 1.618

                  console.log('ema26: ' + ema26)
                  console.log('ema12: ' + ema12)
                  console.log('macd[i]: ' + macd[i])
                  console.log('signal[i]: ' + signal[i])

                  // if MACD < Signal time to sell

                  if (macd[i] < signal[i]) {
                    if (bought) {
                       bought = false
                       sold = true
                       commission += nextPrice * (that.fees / 100)
                       profit += (numCoins * nextPrice - (numCoins * buyPrice))
                       trades++
                       if (numCoins * nextPrice > numCoins * buyPrice) {
                        success++
                       }
                       that.simulationPoints = that.simulationPoints.concat({
                         coin: 'btc',
                         date: dates[i],
                         action: 'Sold',
                         numCoins: numCoins,
                         price: price
                       })
                       that.analysis += '<h3>Sold ' + numCoins + ' ' + 'BTC at ' + price + ' on ' + dates[i] + '</h3>'
                    }
                  }

                  // if MACD > Signal time to buy
                  else if (macd[i] > signal[i]) {
                    if (!bought) {
                      bought = true
                      sold = false
                      buyPrice = price
                      commission += price * (that.fees / 100)
                      numCoins = that.buyingPower / price
                      that.simulationPoints = that.simulationPoints.concat({
                        coin: 'btc',
                        date: dates[i],
                        action: 'Bought',
                        numCoins: numCoins,
                        price: price
                      })
                      that.analysis += '<h3>Bought ' + numCoins + ' ' + 'BTC at ' + buyPrice + ' on ' + dates[i] + '</h3>'
                    }
                  }
                }

                profit -= commission

                that.analysis += '<h3>Total profit was ' + profit + '</h3>'
                that.analysis += '<h3>Total commission taken was ' + commission + '</h3>'
                that.analysis += '<h3>Trades were successful ' + (success/trades)*100 + '% of the time</h3>'
                that.analysis += '<h3>Current support price is ' + support1 + ' and resistance is ' + resistance1 + '</h3>'
                that.analysis += '<h3>Current MACD is ' + macd[dates.length - 1] + ', prepare to sell if dips below '
                + signal[dates.length - 1] + ' and prepare to buy if it goes above it.' + '</h3>'

                const simulationDates = []
                const simulationPrices = []
                that.simulationPoints.forEach((point) => {
                  simulationDates.push(point.date)
                  simulationPrices.push(point.price)
                });

                that.spinnerVisible = false

                that.renderChart('BTC', dates, prices)
                that.renderSimulation('BTC', simulationDates, simulationPrices)
        }
      })
      }
    }
  }
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
header {
  display: flex;
  justify-content: center;
}

.main {
  margin: 0 20%;
}
</style>
