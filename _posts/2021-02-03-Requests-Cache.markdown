---
layout: post
title:  "Faster Backtesting with requests-cache"
date:   2021-02-03 00:00:00 -0500
categories: code
---

# The Problem
Lately, I've been playing with algorithmic trading using [Backtrader](https://www.backtrader.com/), which is a spectacular tool for experimenting with algorithmic trading strategies. This hobby could be considered a problem in itself, but that's not today's topic. A strategy I've been testing involves maintaining a list of S&P500 stocks, ranking them by some metrics, and making a portfolio out of the top 20% or so. Essentially, a really impersonal buy-and-hold strategy.

Unfortunately, this means I need to collect data on 500 stocks over my backtesting time range (5+ years). I would really prefer to use Backtrader's automatic Yahoo Finance data source `bt.feeds.YahooFinanceData`, but I don't want to send 500+ requests to Yahoo every time I tweak a single variable. On a list of just 16 stocks, it takes ~40 seconds to query 5 years of data and run the strategy. There has to be a better way.

# Enter `requests-cache`!
Backtrader supports proxies (so we could set up a caching proxy), but under the hood, backtrader's [Yahoo feed](https://github.com/mementum/backtrader/blob/master/backtrader/feeds/yahoo.py) uses the `requests` module! This means we can solve our problem without even touching a proxy service or having to set up local SSL certificates to cache HTTPS.

All we need to do is run `pip install requests-cache` and add two lines of code to the start of our Backtrader script.

{% highlight python %}
import requests_cache
requests_cache.install_cache('test_cache', expire_after=3600)
{% endhighlight %}

Now, requests-cache will automatically cache any requests made via the python `request` module in a file called `test_cache.sqlite`. Even HTTPS. With this little change, backtesting against 16 stocks over 5 years finishes in 10 seconds: an improvement of 75%!