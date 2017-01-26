---
layout: single
title: "The Unofficial Guide to Tableau's Vizportal API"
date: 2017-01-26
tags: [APIs, Tableau, Tableau Server]
---

Late last year I began [experimenting](https://viziblydiffrnt.github.io/blog/2016/12/14/tableaus-undocumented-api-made-easy-with-python) with Tableau's Undocumented API and I thought:

> "Wow this was hard to figure out! 

> What if there was reference out there with all of these methods in it so other people didn't have to dig through their browser's code or run network traces and inspect the logs to see how these calls work?"

Good news! Now there is...

Presenting the Unofficial Guide to Tableau's Vizportal API. My goal with this guide was to capture the majority of the calls you can perform using the Vizportal API and make it accessible to anyone. While Tableau offers a pretty robust REST API, there are a few things that haven't made it in there that would be really useful (like triggering an extract refresh). For now, you can experiment with the Vizportal API.

This guide is by no means **complete** and I'm sure as Tableau adds new functionality there will be updates to be made. Consider this **v0.1** with more to come in the future. 

<iframe src="https://viziblydiffrnt.github.io/vizportal.html" frameborder="0" allowfullscreen="true" height="1000" width="1000">
</iframe> 