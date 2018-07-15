+++
author = "Brandon Nicoll"
date = 2016-02-05T01:21:34Z
description = ""
draft = false
slug = "octo-progression"
tags = ["Octopus Deploy"]
title = "Querying Release Progression from the Octopus Deploy API"
image = "/OctopusProgressionHeader.png"

+++

<img src="/OctopusProgressionHeader.png" style="max-width: 100%" />
I built a tool this week that required retrieving release progression information from the Octopus Deploy API. While this information *is* exposed through the API, as of this writing, the [API documentation](https://github.com/OctopusDeploy/OctopusDeploy-Api/wiki/Progressions) is lacking for this functionality, and the [Octopus.Client](http://docs.octopusdeploy.com/display/OD/Octopus.Client) library doesn't provide an out-of-the-box way to query progressions.

### "Paste JSON as Classes" (aka Cheating)
In order to get some objects to deserialize the Octopus JSON reponse into, first I queried my Octopus Server for a past release's progression. This can be found at **{Your Octopus Server}/api/releases/{release-id}/progression**. I copied the JSON output, created a new C# class, and made use of Visual Studio's "Paste JSON as Classes" feature.
![](/PasteJSONAsClasses.png)
This saved a lot of time as the JSON output from this API was fairly large and complicated. The pasted root object was aptly named "Rootobject" by Visual Studio. I changed that class name to **OctopusProgressionResponse** and left the other class names as generated.

### Querying Progressions
In my situation, I know the Octopus Release ID ahead of time, so I'm able to query for progressions with code that looks something like this:

```prettyprint lang-csharp
var octoServer = ConfigurationManager.AppSettings["OctopusServer"];
var apiKey = ConfigurationManager.AppSettings["OctopusAPIKey"];
var octoEndpoint = new OctopusServerEndpoint(octoServer, apiKey);
var octoRepo = new OctopusRepository(octoEndpoint);
var octoRelease = octoRepo.Releases.FindOne(x => x.Id == SOMEID);
var octoProgression = new OctopusProgressionResponse();
if (octoRelease.HasLink("Progression"))
{
    var progression = octoRelease.Links.Where(x => x.Key == "Progression").FirstOrDefault();
    var httpClient = new HttpClient
    {
        Timeout = new TimeSpan(0, 0, 20),
        BaseAddress = new Uri(ConfigurationManager.AppSettings["OctopusServer"])
    };
    httpClient.DefaultRequestHeaders.Add("X-Octopus-ApiKey", ConfigurationManager.AppSettings["OctopusAPIKey"]);
    var response = httpClient.GetAsync(progression.Value).Result;

    if (response.IsSuccessStatusCode)
    {
        var responseContent = response.Content;
        var responseString = responseContent.ReadAsStringAsync().Result;
        octoProgression = JsonConvert.DeserializeObject<OctopusProgressionResponse>(responseString);
    }
}
```

### Conclusions
It's a bit of a let-down that this functionality is hidden so deeply, as it can be very useful while building tools that work in tandem with Octopus to keep track of where different versions of your software are at in the release process. The good news is that with a little bit of extra effort, we are able to extract this information from the Octopus API. Overall, I've been very impressed with the Octopus Deploy dev team's efforts to be "[API-first](http://docs.octopusdeploy.com/display/OD/Octopus+REST+API)"; using the Octopus.Client library is mostly a breeze. 