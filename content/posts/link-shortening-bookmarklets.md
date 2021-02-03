---
title: "Link Shortening Bookmarklets"
date: 2019-04-26T23:18:53Z
draft: false
---

Amazon and eBay love to use ugly links which aren't nice to share. Here's two bookmarklets to clean up these URLs before copying them.

## How to use

Create a bookmark in your browser of choice, copy and paste the below code into the URL part of the bookmark, give it a witty name and save.

Tested with ebay.co.uk, ebay.com, ebay.de, amazon.co.uk, amazon.com in Chrome

## eBay

```javascript
javascript:(function(s){var l = /(.*ebay\..*itm.).*\/(\d+).*/.exec(location); prompt('Short URL', l[1]+l[2])})()
```

# Amazon

```javascript
javascript:(function(s){var l = /(.*amazon\..*?\/).*(dp\/.*?\/).*/.exec(location); prompt('Short URL', l[1]+l[2])})()
```

