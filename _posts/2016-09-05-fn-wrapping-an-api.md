---
layout: field-note
title:  "Anything you don't own wrap in a class or adapter"
date:   2016-08-18 19:35:09 -0500
type: "field-note"
tags: [testing,api]
---

[upcase.com][upcase] Weekly iteration on testing APIs

Ideally if you are interacting with the outside world either through APIs or outside gems you should be putting your connection to it into a wrapper or adapter class. This creates an internal API for the outside source you can control. So if you want to substitute the what is in the wrapper or if the outside API changes you have a central place to change or fix.


[upcase]: https://thoughtbot.com/upcase/ 
