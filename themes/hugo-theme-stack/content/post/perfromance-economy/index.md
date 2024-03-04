+++
author = "Vladimír Urík"
title = "Performance Economy"
date = "2024-03-03"
description = "How I made economy plugin for Minecraft Network more efficient"
tags = [
 "java",
 "bukkit",
 "velocity",
 "mysql",
 "redis",
 "minecraft"
]
categories = [
 "Engineering",
 "Minecraft",
]
series = ["Minecraft Development"]
aliases = ["performance-economy"]
image = "steve-johnson-YJGq5H9ofy0-unsplash.webp"
+++

## Introduction 
Most Minecraft networks/servers have an economy plugin. Mostly the currencies are something like "Coins", "Gems", "Money". So it's from the perspective of the user, which is basically always the same. They see some kind of currency and it's hard to guess exactly how it's working in the background. On smaller servers it doesn't really matter. But when you have a Minecraft network, it's good to know how your global economy works, and to have it as optimized and "foolproof" as possible, so that you do not get various unintentional bugs of duplication, unintentional deletion, or general data loss, and so that the economy does not lock up various services that are essential to your network. Such as a database.

## The Problem
Most economy plugins on the market work on the following principle, as I mentioned before. When a player connects to a server on the network, the plugin sends a query to get the user's data, then if the user doesn't exist, it sends another query (not only when the player connects, as the currency count also needs to be updated). Most of these plugins are still working in sync, so they are "lagging" the server (because the Minecraft plugins are working on the main thread of the server). First of all, we need to consider that Economy is not the only plugin that is pulling from the database when it joins and updates data. As I wrote, on smaller, single-server Minecraft servers, this is sufficient. But if you have a network and the player is connecting between the servers (thanks to a proxy, in our case Velocity), the number of requests to the database can increase quite a bit.

![How Market Plugins Works](how-market-plugins-works.png)

## The Solution
To solve this problem I needed something that could share the cache between servers on the network. I thought about using an http server. But that would be slow enough for my solution. That's why I chose Redis.

> Redis is the open-source, in-memory data store used by millions of developers as a cache, vector database, document database, streaming engine, and message broker

But for my solution, implying redis is not enough. The economy has to be bulletproof. It can't fail and it can't crash the server. That's why I made it asynchronous. (In other words, its functionality doesn't work on the main server thread.) This is why I have proposed the following "double" functionality. The economy plugin will behave differently on proxies and differently on servers. You might ask why. When a player disconnects from a server, this doesn't mean that they have disconnected from the network. He could simply move to a different server that is registered under a proxy. Also, I chose keydb (a fork of redis). It is many times more efficient and faster than redis.

![Redis vs KeyDB](redis-vs-keydb.png)
![Redis vs KeyDB Latency](redis-vs-keydb-latency.webp)

Back to how the plugin works. When a player connects to the proxy, their data is loaded asynchronously from the database. It is stored locally in the proxy cache (for each player's economy handling), but also in redis. When the player then connects to the server under the proxy, it will try to load the data from redis 3 times (in case it slows down for some reason, or in case there is a large rush of players). If the data is not found in redis or the redis connection fails, the plugin pulls the data from the database and caches it locally on the server (everything outside the server's main thread). This allows the economy to run smoothly even if the redis connection fails. When we do changes to the player's economy, any changes will be asynchronous. When the player disconnects from the server, the data is flushed out of the server's cache, but it remains in the redis. It is only removed from the redis when the player disconnects from the network, i.e. when it disconnects from the proxy.

## Conclusion
The economy plugin is now a lot more efficient and a lot faster. It doesn't slow down the server. It doesn't slow down the database. Because I use secure SQL queries, it's also much more reliable and doesn't lose any data.

![How it works on servers](server-schema.png)
![How it works on proxies](proxy-schema.png) 

## References
- KeyDB benchmark images and data [KeyDB is a Fork of Redis that is 5X Faster](https://medium.com/@tedmandarin/keydb-is-a-fork-of-redis-that-is-5x-faster-164757232bac)
- Why I choose redis instead of MySQL? [MySQL as Redis vs Redis?](https://dkomanov.medium.com/mysql-as-redis-vs-redis-74b788af9c6f)

## Acknowledgements
- Photo by [Steve Johnson](https://unsplash.com/@steve_j?utm_source=Vladimir-Urik&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=Vladimir-Urik&utm_medium=referral)
