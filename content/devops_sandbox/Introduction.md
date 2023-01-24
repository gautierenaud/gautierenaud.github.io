---
title: 01 Introduction
fileName: 
tags:
- DevOps
categories:
date: 2023-01-24
lastMod: 2023-01-24
---
Hi, this is my personal sandbox project, where I'll try to learn about different part that I attribute (maybe wrongly) to DevOps. I'm more of a Dev than an Ops, so I wanted some time/playground to learn a bit more about the concepts that are dealt with on the Ops side.

The idea would be to learn about (thrown in random order, not exhaustive nor contractual):
* observability
* traceability
* idempotency/replayability
* chaos engineering
* logs
* sidecars
* profiling
* security
* repo structure
  * monorepo
  * by component
  * split deployment

In order to learn all those different topics, I'll try to create a dummy project with a simple architecture (some services, a bit of Pub/Sub buses and a database) and enhancing it bit by bit with the different concepts as I learn about them. I just hope I wont have to start over, as I realize some topics need to be taken into account since the inception of a project (and as an engineer I know nothing will stick to the initial plan). I'll probably use [k3s](https://k3s.io/) in order to have something that will look like a real life application (emphasis on "look like").
