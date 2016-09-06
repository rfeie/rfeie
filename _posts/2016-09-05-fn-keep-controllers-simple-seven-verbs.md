---
layout: post
title:  "Keep Controllers Simple, ideally you should restrict yourself to the seven basic verbs"
date:   2016-08-18 19:35:09 -0500
type: "field-note"
tags: [ruby-on-rails,clean-code]
---

[upcase.com][upcase] TDD Rails Trail

This is a short-hand rule or code smell in Rails to avoid putting too much responsibility in the controller. Any Model related code should go into the model. And if you have a lot of extra responsibility in a controller consider refactoring that responsiblity into another controller.

[upcase]: https://thoughtbot.com/upcase/ 
