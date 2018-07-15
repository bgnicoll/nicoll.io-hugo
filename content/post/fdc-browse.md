+++
author = "Brandon Nicoll"
categories = ["Redis"]
date = 2017-02-26T06:07:25Z
description = ""
draft = false
image = "/BrowseHeaderTwitter-2.png"
slug = "fdc-browse"
tags = ["Redis"]
title = "Building the Fonts.com Browse Filters Using Redis"

+++

<img src="/BrowseHeader.png" style="max-width: 100%" />
Recently on fonts.com, we released a new way to browse and discover fonts. We wanted this tool to be easy to use and return results quickly. In order to accomplish the latter, I made heavy use of Redis data structures and built-in commands. You can find the finished product at [fonts.com/browse](https://www.fonts.com/browse). The book *Redis in Action* by Dr. Josiah Carlson helped out immensely while building this feature. Specifically, Chapter 7 - *Search-Based Applications*. I would recommend picking up this book to anyone looking to use Redis in their application. 

## Building Inverted Indexes
Results returned by the filters are on the font family level. For those unfamiliar, a family is a grouping of fonts of the same typeface. Fonts within a family vary by things like italic versions, bold versions, heavy weights, or light weights.

![](https://cdnimg.fonts.net/CatalogImages/25/5390783.png)
<label style="text-align: center; display: block;">**Figure 1.) The 18 fonts making up the Neue Kabel family**</label>
 For the filters to work properly, the system needs to look at all fonts in a family and return that family as a result if one or more fonts has that corresponding characteristic. 

I chose to use the Redis SET data structure ([https://redis.io/topics/data-types#sets](https://redis.io/topics/data-types#sets)) to hold family IDs that all contain the same characteristic. This concept is called an [inverted index](https://en.wikipedia.org/wiki/Inverted_index). The time consuming thing here was coming up with SQL queries for each filter option.

<img src="/SetExample-3.png" alt="" style="display: block;margin: auto;">

<label style="text-align: center; display: block;">**Table 1.) Example visualization of Redis SETs storing family IDs**</label>

## Sort Ordering

As you may have noticed, Redis SETs are *unordered* lists of strings. The browse tool has four sorting options. 

<img src="/SortOptions.png" alt="" style="display: block;margin: auto;">
<label style="text-align: center; display: block;">**Figure 2.) Sorting options**</label>


Luckily, we have another Redis data structure that can help. ZSETs ([https://redis.io/topics/data-types#sorted-sets](https://redis.io/topics/data-types#sorted-sets)) are sets that contain one additional piece of information, a numerical score. This data structure allowed me to pre-compute the complete order each family should appear in. 

<img src="/SortZSETs-2.png" alt="" style="display: block;margin: auto;">
<label style="text-align: center; display: block;">**Table 2.) Example visualization of Redis ZSETs storing sorted family IDs**</label>

### Intersects and Unions
When you click various filter options, the system combines the corresponding sets with each other and sorts them according to the selected sort option. The Redis command ZINTERSTORE ([https://redis.io/commands/zinterstore](https://redis.io/commands/zinterstore)) will combine n number of SETs together with logical AND operations, then sort them according to the ZSET scores, and finally create a new ZSET with the ordered results.

<img src="/ZInterstoreExample-3.png" alt="" style="display: block;margin: auto;">
<label style="text-align: center; display: block;">**Figure 3.) ZINTERSTORE example**</label>

For the most part, this is straightforward, but one filter in particular causes some trouble.

<img src="/PriceSlider.png" alt="" style="display: block;margin: auto;">
<label style="text-align: center; display: block;">**Figure 4.) Price slider**</label>

The other filter options such as languages, properties, and classifications are all inclusive (logical AND), but the pricing options are exclusive (logical OR). Redis doesn't have the capability to combine logical ANDs with logical ORs in the same command, so before applying the other filters I had to use SUNIONSTORE (https://redis.io/commands/sunionstore) with each selected price filter to create a single price SET with the desired options. 

## Paging
The result of the previous section is a ZSET containing the entire sorted list of the combined filter options. So how do we get a smaller subsection of this list to render a page? How do we know how to render the page count?

<img src="/PagingAndNumberOfResults-1.png" alt="" style="display: block;margin: auto;">
<label style="text-align: center; display: block;">**Figure 5.) Number of results and pages**</label>

We can get the total number of items in a ZSET from the ZCARD command (https://redis.io/commands/zcard). We can get the desired page's family IDs by making use of the ZRANGE command (https://redis.io/commands/zrange)

## Presentation Data
After executing ZRANGE, we're left with an ordered subsection of family IDs. This is obviously not enough to render a page full of results. Part of the daily job that refreshes this data also retrieves and stores enough information for us to display each result. This meta data is stored in a Redis data structure called a HASH (https://redis.io/topics/data-types#hashes). HASHes are like mini-versions of Redis, containing n number of key-value pairs.

<img src="/HGETALL-1.png" alt="" style="display: block;margin: auto;">
<img src="/KabelPresentationData.png" alt="" style="display: block;margin: auto;">
<label style="text-align: center; display: block;">**Figure 6.) Example of presentation data stored in Redis HASH**</label>

Depending on your needs, you could also store this type of object in a plain string and serialized in JSON, [MessagePack](http://msgpack.org/), [Protocol Buffers](https://developers.google.com/protocol-buffers/), and so on. I chose a HASH in this instance so that individual fields could be accessed and updated without the need to read the entire object, make the modification, and write it back. Most of the presentation data changes very rarely, but we show an "On Sale" flag for families that are (you guessed it) on sale. Since promotions can come and go at any time, I have a job running every 10 minutes to make sure these values are up-to-date.

## Dynamic Preview
Dynamic preview is what we call the preview style changing depending on which filter options you've selected. As I mentioned earlier, the results are displayed on a family level and reflect all fonts in a typeface, but we can only display a single style for the preview. Ordinarily when faced with this problem, we display the family preview in the "normal" weight, or *Roman* weight in typography terms. 

With the release of this tool, certain filter options will change the preview style. Select "Monospaced", any "Width" option, or any "Weight" option and we try to show you the style you might be looking for as the preview style.

I store a little bit of serialized data for each font in each family in Redis LISTs(https://redis.io/topics/data-types#lists). If you select any of the aforementioned filter options, I have logic in place to attempt to match your filter with an appropriate preview style. 
<img src="/LRANGE-2.png" alt="" style="display: block;margin: auto;">
<img src="/DynamicPreview.png" alt="" style="display: block;margin: auto;">
<label style="text-align: center; display: block;">**Figure 7.) Example of LRANGE usage and serialized dynamic preview meta data**</label>



## Results
After all this planning and research, it's nice to know that the hard work is paying off. Not only is the tool a success with users, but we're seeing very low latencies from the microservice I built to handle these requests. 
![](/BrowseLatency.png)
<label style="text-align: center; display: block;">**Figure 8.) Seven day average latency data from New Relic**</label>

This feature has been my all-time favorite piece of software to write and it is certainly the thing I'm most proud of in my career. I couldn't have done it without the help of my teammates, [Piper Lawson](https://twitter.com/uxpiper) (UX\UI) and [Reed Rizzo](https://twitter.com/reedling78) (Front End Developer). Without them, this feature would have looked something like this:
```prettyprint lang-json
{
  "TotalResults": 20829,
  "Families": [
    {
      "FamilyId": 1245395,
      "FamilyName": "Neue Helvetica®",
      "FamilyURL": "font/linotype/neue-helvetica",
      "OnSale": false,
      "NumberOfStyles": 59,
      "LicenseAvailability": 95,
      "FoundryName": "Linotype",
      "FoundryUrl": "font/linotype",
      "FontFileMd5": "5a1d7e236d9bfb682fe593ff4b8608bd",
      "InMLS": true
    }
  ]
}
```
<label style="text-align: center; display: block;">**Figure 9.) Nobody wants to buy fonts this way**</label>

We hope you enjoy using this tool as much as we enjoyed building it!