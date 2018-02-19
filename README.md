## Crypto Simulator Web App

This App is a Node / VueJS App that I created that ports the same functionality of the Stock Simulator React App that
I created in 2017 except for crypto, and using VueJS. Instead of using buy and sell limits,
it uses MACD and RSI as indicators to buy or sell.

It uses the `http://coindesk.com/` API to get the BPI data.

## Improvements

1) Add support for more coins. Right now Coindesk only allows you to get historical BTC data but as soon as coinmarketcap
opens up their historical API I can use that.

2) Incorporate other indicators besides MACD and RSI like Volume, Ichimoku, Candlesticks, Renko, Bollinger bands etc

## Instructions for running

First run `npm install` to install all the node nodules.

Run `npm start` from the root to build the app and get the server up and running.
By default it starts up on `http://localhost:8080/`.

-Tong Zou
