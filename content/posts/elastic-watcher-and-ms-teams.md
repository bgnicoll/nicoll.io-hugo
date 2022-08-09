+++
author = "Brandon Nicoll"
date = 2022-08-08T23:13:25Z
description = ""
draft = false
image = "/ElasticWatcherTeamsHeader.png"
slug = "elastic-watcher-and-ms-teams"
tags = ["Elasticsearch", "ChatOps", "Monitoring", "Logging"]
title = "Building a Microsoft Teams Integration with Elastic Watcher"
thumbnail = "/ElasticWatcherTeamsHeader.png"

+++


<img src="/ElasticWatcherTeamsHeader.png" style="max-width: 100%" />
Logging and monitoring: two of my favorite pastimes. Throughout my career, many tools to get the job done have come and gone. I could wax poetic and reminisce for hours about all the times I tried to keep a cool head while scanning logs for *any* hint as to why my software wasn't working the way I expected. Today, I added one more tool to my toolkit and I'd like to share my experience. Mostly because I had a surprising amount of trouble figuring out how to do it.

### The Idea
The idea was simple: scan logs for specific events and when they happen, get more detailed information about those events in front of the right people. We're running a standard ELK stack and I've used <a href="https://www.elastic.co/guide/en/kibana/current/watcher-ui.html">Elastic Watcher</a> in the past for a more rudimentary scenario. This time I wanted to extract data from relevant log events in order to create <a href="https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-reference#office-365-connector-card">Microsoft Teams Connector Cards</a>, then send those cards to a dedicated channel via <a href="https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook">Incoming Webhook</a>. One of the tricks that made this easier was use of <a href="https://www.elastic.co/guide/en/beats/filebeat/current/decode-json-fields.html">Filebeat's Decode JSON fields processor</a> to enable structured logging.

### The Log
I made things easier for myself by introducing a boolean to the log messages of the events I was interested in. In the example below, it's called ```raiseAlarm```:

{{< highlight csharp >}}
try
{
    await thingIHopeWorks();
}
catch (Exception ex)
{
    _logger.LogError("thingIHopeWorks() did not work. raiseAlarm: {raiseAlarm} | customerId: {customerId}", true, customerId)
}
{{< / highlight >}}

### The Superfluous Details I'm Skipping
At this point I'm going to take a shortcut and go straight to the exact syntax of the watch, but I will list some useful links that got me to there. (Some of these appeared earlier in the article)

<ul><a href="https://docs.microsoft.com/en-us/dotnet/core/extensions/logging">.NET Logging</a></ul>
<ul><a href="https://serilog.net/">Serilog</a></ul>
<ul><a href="https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html">Elasticsearch</a></ul>
<ul><a href="https://www.elastic.co/guide/en/kibana/current/index.html">Kibana</a></ul>
<ul><a href="https://www.elastic.co/beats/filebeat">Filebeat</a></ul>
<ul><a href="https://www.elastic.co/guide/en/beats/filebeat/current/decode-json-fields.html">Filebeat Decode JSON Fields</a></ul>
<ul><a href="https://www.elastic.co/guide/en/kibana/current/watcher-ui.html">Elastic Watcher</a> (scroll to the "advanced" section)</ul>
<ul><a href="https://docs.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook">Microsoft Teams Incoming Webhook</a></ul>
<ul><a href="https://docs.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/cards-reference#office-365-connector-card">Microsoft Teams Connector Card</a></ul>

### The Teams Connector Card
I crafted the card JSON outside of the watcher JSON. It will be obvious why I did this soon enough. Here is a reasonable facsimile of the finished product:

{{< highlight json >}}
{
  "@type": "MessageCard",
  "@context": "http://schema.org/extensions",
  "themeColor": "7787E9",
  "summary": "{{ctx.payload._source.environment}} important event",
  "sections": [
    {
      "activityTitle": "{{ctx.payload._source.environment}} Important Event",
      "activitySubtitle": "Customer ID: {{ctx.payload._source.customerId}}",
      "activityImage": "https://cdn.nicoll.io/an-image.png",
      "facts": [
        {
          "name": "Environment",
          "value": "{{ctx.payload._source.environment}}"
        },
        {
          "name": "Timestamp",
          "value": "{{ctx.payload._source.timestamp}}"
        },
        {
          "name": "Log message",
          "value": "{{ctx.payload._source.message}}"
        }
      ],
      "markdown": false
    }
  ],
  "potentialAction": [
    {
      "@type": "OpenUri",
      "name": "Open Log Event",
      "targets": [
        {
          "os": "default",
          "uri": "https://my-kibana-instance.example.com/app/discover#/doc/index-pattern-id/{{ctx.payload._index}}?id={{ctx.payload._id}}"
        }
      ]
    }
  ]
}
{{< / highlight >}}

The important thing to note here is the usage of the ```{{ctx.payload.blah}}```. This is how we tell Elastic watcher which fields\properties from the log events to pipe in as variables.

### The Watch
Thanks for making it this far, here is where my rant begins. 

I believed this was going to be a simple task that many developers would find useful. I also believed that many developers before me would have worked this out and information on how to replicate their success would be plentiful. However, the Elastic documentation, their question and answer forums, Stack Overflow, and almighty Google itself failed me in this endeavor. So, I spent several days bashing my head against this wall and now I'm attempting to give back. Here is another reasonable facsimile of the finished product:

{{< highlight json >}}
{
  "trigger": {
    "schedule": {
      "interval": "15m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [
          "my-index-*"
        ],
        "body": {
          "query": {
            "bool": {
              "filter": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-15m"
                    }
                  }
                },
                {
                  "match": {
                    "raiseAlarm": true
                  }
                }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gte": 1
      }
    }
  },
  "actions": {
    "importantTeamsWebhook": {
      "foreach": "ctx.payload.hits.hits",
      "webhook": {
        "method": "POST",
        "path": "/the-rest-of-the-webhook",
        "body": "TEAMS CARD JSON GOES HERE",
        "host": "my-company.webhook.office.com",
        "scheme": "https",
        "port": 443
      }
    }
  }
}
{{< / highlight >}}

##### Card JSON Removal From Above Example
First, I will note that in order to get the Teams card JSON into ```actions.importantTeamsWebhook.webhook.body```, I ran the previous JSON through a whitespace removal tool and also a string escape tool. This was, of course, over 1000 characters on a single line so it was omitted.

##### URL
Next, I'm going to rant about the webhook URL. Yes, Elastic makes you split up the webhook URL into these asinine parts and, yes, it was a few tries before I got this right. Here are some of the reasons why:

The <a href="https://www.elastic.co/guide/en/elasticsearch/reference/current/actions-webhook.html#_webhook_action_attributes">watcher webhook action attributes</a> lists the <code>url</code> attribute as 
> *"A shortcut for specifying the request scheme, host, port, and path as a single string. For example, http://example.org/foo/my-service."*

Naturally, I opted for this route at first and left out path, host, scheme, and port. I clicked save and was met with an error telling me that <code>host</code> and <code>port</code> are required fields. None of their examples included <code>scheme</code> so I initially brushed past that as well and that cost me a few minutes.

##### ctx.payload
Of all the time I spent trying to get this to work, the majority of it was spent trying to navigate and understand the syntax ```ctx.payload.hits.hits``` array. A key point that eluded me for some time is that many the log's fields are hidden behind the ```_source``` property. I also found many different ways to access individual elements of the array. ```hits.hits.0```, ```hits.hits[0]```, ```hits.hits["0"]``` none of which seemed to work at first. The internet was parched for solid examples on what to do here. Eventually, I stumbled on the ```foreach``` property that will allow the webhook action to take place for every log hit coming back from the query. Sending many Teams webhooks requests in a short amount of time simply will not scale. I would only recommend this for low throughput use cases.

##### Creating a Kibana Link
This was also something that was more difficult than it should have been. I should be able to construct a Kibana link directly to a single document given the document contains both the index it lives in and its unique id. For some reason, I also need this ambiguous "index pattern id". Try Googling "construct kibana URL to single document". The results are disappointing.

##### Watcher Simulator Tool
There's a watcher simulator tool in Kibana that I just could not figure out. It's meant for debugging and I probably didn't read the right documentation or whatever, but it was not intuitive at all. I had to inject one of the logs (easy enough) and modify the ```range``` filter longer and longer (kind of lame) for debugging.

### The Conclusion
I finally got this to work over (embarrassingly enough) several days, but I am quite happy with the results. The watch and the neatly organized card it creates when fired will prove invaluable for my intended use case. It was just a long, painful walk to get there.