+++
author = "Brandon Nicoll"
date = 2013-07-08T16:56:00Z
description = ""
draft = false
slug = "convert_dictionary_int_bool_to_dictionary_int_enum_with_enumerable_to_dictionary_"
title = "Convert Dictionary<int, bool> to Dictionary<int, enum> With Enumerable.ToDictionary()"

+++

I'm getting some data in the form of a string as a key value pair collection of integer IDs and a boolean value(a status represented on the page by a checkbox). First, I call DeserializeObject() from the Json.NET library and convert the string to a Dictionary&lt;int, bool&gt;. I can then use the ToDictionary() method and lambdas to make a new Dictionary that translates the boolean field to an enum value. I need the values to be of the enum type, rather than the bool because there are many other statuses possible, but we only wanted to represent toggling between the two most common on the page.  

{{< highlight csharp >}}
    var dict = ((Dictionary<int, bool>)
    JsonConvert.DeserializeObject
    (lessonsStatuses, typeof(Dictionary<int, bool>))).ToDictionary
    (x => x.Key, x => x.Value ? ObjectStatusEnum.Status1 : ObjectStatusEnum.Status42);
{{< / highlight >}}
