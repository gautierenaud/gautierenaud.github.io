---
title: Projects/IdealProjectTemplate
tags:
categories:
date: 2023-01-22
lastMod: 2023-01-22
---
2023-01-01

  + A template that would illustrate several things that I find interesting when deploying services

    + observability

    + traceability

    + idempotency/replayability

    + chaos engineering

    + logs

    + sidecars

    + profiling

    + git structure

      + monorepo

      + by component

      + split deployment

  + maybe a k3s/minikube with a stub that will put data into kafka, a consumer that will do some operations and put it into a db, and a service that would request that db

    + maybe one more layer of service to test circuit breaker?

# Representation

  + if I choose to represent it as a website, I should keep it as simple as possible

    + http://bettermotherfuckingwebsite.com/

    + Markdown to website?

    + export Logseq notes as the website?

      + Publish

        + Will have a Logseq webapp with only the pages marked as "public" accessible

        + https://github.com/sawhney17/logseq-schrodinger

          + exports to [hugo](https://gohugo.io/)

            + #web #static #generate

            + > A Fast and Flexible Static Site Generator

            + Markdown -> HTML

    + export it on github as https://pages.github.com

  + no bloat too?

    + check that images are compressed?

    + no more than X00kb per page?

![IMG_2019-05-08-11550571.jpg](/assets/img_2019-05-08-11550571_1674400134843_0.jpg)
