+++
author = "Brandon Nicoll"
date = 2014-05-15T17:34:00Z
description = ""
draft = false
slug = "hashset"
title = "Unique Collections With HashSet<T>"

+++

I used to say that [Dictionary &lt;TKey, TValue&gt;](http://msdn.microsoft.com/en-us/library/xfhwa508(v=vs.110).aspx) was my favorite C# feature. Dictionary offers great speed and versatility, but there are a few drawbacks that don't make it fit every situation: 

*   Attempting to add a key-value pair of an existing key throws an ArgumentException, causing the need to check first with [ContainsKey()](http://msdn.microsoft.com/en-us/library/kw5aaea4(v=vs.110).aspx)
*   Sometimes I only want a collection of unique things, not key-value pairs of things

HashSet is so appealing to me because it has the speed and uniqueness of Dictionary, but with added flexibility with fewer lines of code. The [HashSet&lt;T&gt;.Add](http://msdn.microsoft.com/en-us/library/bb353005(v=vs.110).aspx) method returns returns a boolean: true if that object was added to the HashSet, false if it already exists. This feature is what gives the HashSet its true power. With it, you can use the HashSet as a lightweight and fast collection of existing objects and the conditional Add() as a check before possibly attempting to modify other collections that would take more time. I've created a quick example of the HashSet in action in an ASP.NET MVC 5 project and pushed it to GitHub. [https://github.com/bgnicoll/HashsetExample](https://github.com/bgnicoll/HashsetExample) I also deployed to Azure for fun. I get 10 free sites and it can pull the repo directly from GitHub, why not? [http://hashset.azurewebsites.net/](http://hashset.azurewebsites.net/)

In the example, I simply created a HashSet&lt;string&gt; and gave the user an input to attempt to add something to it. I output the result to the console. One thing to note is the [constructor](http://msdn.microsoft.com/en-us/library/bb359100(v=vs.110).aspx) contains [StringComparer.OrdinalIgnoreCase](http://msdn.microsoft.com/en-us/library/system.stringcomparer.ordinalignorecase.aspx), which makes the HashSet case-insensitive. Maybe you want this, maybe you don't. 