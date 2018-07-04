---
title:  Using Azure text translation services
author: Will Andrews
date: 2018-06-20
order: 13
---

As part of a discovery task at work, I was asked to take a look at some translation services available. First one I tried was Azure, and I have to say I'm really impressed. There is also a free tier to use so there's no need to pay to try it out!

First you need to get yourself set up with an Azure account and then create a cognitive service for translations.


Once you have an API key, it's time to start coding.

Doing this is quite easy as it uses a http api call. The following code is an example of what is needed.

``` c#
private static async void translateAsync()
{
    string uri = "https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&to=fr";

    string key = "USE YOUR KEY HERE";

    string inputText = "Hello, how are you today";

    Object[] body = new Object[] { new { Text = inputText } };
    string requestBody = JsonConvert.SerializeObject(body);

    //TextTranslationResult result = new TextTranslationResult();
    using (HttpClient client = new HttpClient())
    {
        using (HttpRequestMessage request = new HttpRequestMessage())
        {
            request.Method = HttpMethod.Post;
            request.RequestUri = new Uri(uri);
            request.Content = new StringContent(requestBody, Encoding.UTF8, "application/json");
            request.Headers.Add("Ocp-Apim-Subscription-Key", key);

            HttpResponseMessage response = await client.SendAsync(request);
            string responseBody = await response.Content.ReadAsStringAsync();
            string result = JsonConvert.SerializeObject(JsonConvert.DeserializeObject(responseBody), Formatting.Indented);

            Console.WriteLine(result);
        }
    }
}
```

So what's going on here? 

The URI is simple enough. You can also translate into multiple languages, but chaining parameters on the end. The following will translate into french and spanish:
```
"https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&to=fr&to=es";
```

The rest is just creating a http request to the uri, adding in the api key into the header. The data returned will look something like this:

``` json
[
  {
    "detectedLanguage": {
      "language": "en",
      "score": 1.0
    },
    "translations": [
      {
        "text": "Bonjour comment vas-tu aujourd'hui",
        "to": "fr"
      }
    ]
  }
]
```
That's it. Quite simple. A full implementation of this, including a web page that takes speech and translates it to languages of choice can be found here: https://github.com/willdot/TranslationServices