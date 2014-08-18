---
layout: post
title: "why sqlite or h2 as embedded database"
description: ""
category: tech note
tags: [sqlite,h2,hibernate]
---
{% include JB/setup %}

## feelings when I'm using sqlite/h2 as embedded Hibernate database
  + H2 is gradle/maven friendly, no extra installation requirements
  + in linux world (including Android) sqlite installation is often simple
  + when using Hibernate, the dialect and driver of Sqlite3 is not as stable as H2
  + H2 needs file lock, so using a sql console (both web gui and cli) when running app is not a good idea
  + H2 needs more than one file
  + H2 is faster than sqlite (about 10x faster in some benchmarks, some one my really cares)
  + the cli shell of H2 is weak
  + sqlite cli is very helpful, and its Python api is intuitional
  + if migrate from Java/Hibernate to Python/SQLAlchomy, of course sqlite will be the first choice
  + sqlite column type is flexible
  + case sensible LIKE of sqlite is a bit complex

## my choice:

<!--more-->

Using sqlite if you are not very very care about speed and driver stability.
