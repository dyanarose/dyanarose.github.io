---
layout: post
title: "Interfering with the .NET MVC default model binder"
date: 2015-01-03 17:33:45 +0000
comments: true
categories: [.net, .net mvc]
published: false
---
Model binding in .NET MVC is brilliant when you're posting a simple object from a form that itself was rendered using .NET MVC.

Even slightly complicated objects. Model binding works generally as expected. But if you want to do something slightly different, like post to a different model than used to create the form fields, or post a non-0 based, non-sequential collection, or post a form not created using .NET MVC at all? Then you need to know how to manipulate the model binding.

There's a lot you can do without having to create custom model binders. I, personally, try to avoid creating custom binders if I can help it. Reason being that the default binder does a lot of boiler plate work. I don't want to have to duplicate any of it, or lose any of it when I don't have to.

