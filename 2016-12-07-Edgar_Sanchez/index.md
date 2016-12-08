
# Describing images in your favorite human language #

*All text and code copyright (c) 2016 by Edgar Sánchez. Used with permission.*

*Original post dated 2016-12-07 available at http://cogitoergofun.ghost.io/describing-images-in-your-favorite-human-language/*

**By Edgar Sánchez**



Several months ago Microsoft published a [moving video](https://www.youtube.com/watch?v=R2mC-NUAmMk) about Saqib Shaikh love for talking computers,
by all means watch it but here is the technical gist, at least as far as this post is concerned: cool sunglasses that take a picture and then talk back to you describing the image. 

![Saqib Shaikh, software engineer and talking computers fan ](saqib.jpg)


As an engineer, it was impossible not to ask yourself 'How did they do it?' and shortly after, as an F# fan, 'Can I do it in F#?'.
It turns out that all of the heavy work can be done using [Cognitive Services](https://www.microsoft.com/cognitive-services), a set of APIs for [vision](https://www.microsoft.com/cognitive-services/en-us/computer-vision-api),
[translation](https://www.microsoft.com/cognitive-services/en-us/translator-api), and [speech](https://www.microsoft.com/cognitive-services/en-us/speech-api).
These cloud services expose themselves as REST services, callable through plain HTTP, albeit with slightly different details.
To begin with here is a function that takes an image URL and returns its description in English:

```fsharp
type VisionApi = JsonProvider< @"resources\visionApiResult.json" >  
let analyzeImage subscriptionKey imageUrl =  
    let visionApiRequest = "https://api.projectoxford.ai/vision/v1.0/analyze?visualFeatures=Description,Tags"
    let response = Http.RequestString(
                    url = visionApiRequest, 
                    headers = [ "ocp-apim-subscription-key", subscriptionKey; ContentType Json ], 
                    body = (imageUrl |> sprintf """{"url": "%s"}""" |> HttpRequestBody.TextRequest) )
    let jsonObject = VisionApi.Parse response
    jsonObject.Description.Captions.[0].Text
```
    
`Http.RequestString()` is part of the [HTTP Utilities](https://fsharp.github.io/FSharp.Data/library/Http.html) at the [FSharp.Data library](https://fsharp.github.io/FSharp.Data/) project;
to use the Vision service you must [get a subscription key for free here](https://www.microsoft.com/cognitive-services/en-us/sign-up). The service returns a JSON string that looks like this:

```
{
    "tags": [
        {
            "name": "sky",
            "confidence": 0.998857855796814
        },
        {
            "name": "outdoor",
            "confidence": 0.99455726146698
        },
        {
            "name": "grass",
            "confidence": 0.9855617880821228
        }
    ],
    "description": {
        "tags": [
            "outdoor",
            "grass",
            "mountain"
        ],
        "captions": [
            {
                "text": "a herd of animals standing on top of a mountain",
                "confidence": 0.38063336153125649
            }
        ]
    },
    "requestId": "17670d1b-be99-496f-b4a0-987aa3bb93e8",
    "metadata": {
        "width": 900,
        "height": 500,
        "format": "Jpeg"
    }
}
```

The description is buried deep in the first item of the `captions` property. As you might know, diving through a JavaScript string could become
a tricky exercise but the [JSON Provider](`https://fsharp.github.io/FSharp.Data/library/JsonProvider.html) comes to the rescue,
`type VisionApi = JsonProvider< @"resources\visionApiResult.json" >` basically takes a sample JSON and generates an F# class mimicking its structure,
with the help of this class getting the description is just a matter of `jsonObject.Description.Captions.[0].Text`
- and of course you get type safety and Intellisense at development time, it doesn't get any easier :-). 

With the text in hand, let's translate it from English to, say, Spanish. Time to use the [Translation API](https://www.microsoft.com/cognitive-services/en-us/translator-api/documentation/TranslatorInfo/overview),
one hurdle here is that the service requires a token which is valid for ten minutes, so you first need to get the token using [another free subscription key that you get here](https://docs.microsofttranslator.com/text-translate.html) - see step 12.


```fsharp
let getCognitiveAuthToken subscriptionKey =  
    let cognitiveTokenRequest = "https://api.cognitive.microsoft.com/sts/v1.0/issueToken/"
    let response = Http.RequestString(
                    url = cognitiveTokenRequest,
                    headers = [ "ocp-apim-subscription-key", subscriptionKey],
                    body = TextRequest "" )
    response
```
    
Once you have a temporary token, we can translate from English to any of several languages using this function:

```fsharp
type TranslationXml = XmlProvider< """<string xmlns="http://schemas.microsoft.com/2003/10/Serialization/">¡Hola mundo!</string>""" >  
let translateText authToken fromLang toLang text =  
    let translateRequest = "https://api.microsofttranslator.com/v2/http.svc/Translate"
    let response = Http.RequestString(
                    url = translateRequest,
                    query = [ "text", text
                              "from", fromLang
                              "to", toLang ], 
                    headers = [ "Authorization", "Bearer " + authToken ] )
    TranslationXml.Parse response
```    

The creators of this service decided to return XML -don't you just love the real world of mostly autonomous teams? ;-)
To extract the translated text from the XML string we can use the [XML Provider](https://fsharp.github.io/FSharp.Data/library/XmlProvider.html), probably overkill here since the XML is so simple
but hey I am trying to show off the FSharp.Data library! :-D 

The final step is converting the now translated text to speech, for this let's use the [Text to Speech Service](https://www.microsoft.com/cognitive-services/en-us/speech-api),
do note this service can use the same temporary token from the previous service. A sample function for going from text to speech is:

```fsharp
let toSpeech authToken lang text =  
    let toSpeechRequest = "http://api.microsofttranslator.com/V2/Http.svc/Speak"
    let response = Http.Request(
                    url = toSpeechRequest, 
                    query = [ "text", text; "language", lang ],
                    headers = [ "Authorization", "Bearer " + authToken ] )
    match response.Body with
    | HttpResponseBody.Binary bytes -> Some bytes
    | _ -> None
```
    
Most of the time the service returns the speech binary representation in the HTTP body but sometimes it may fail so the function returns a `byte[] option`.
And with this, we are done with the vision, translation, and speech cognitive services, all that remains is to play the sound bytes, at least in Windows you can do it like so:

```fsharp
let play (maybeBytes: byte[] option) =  
    match maybeBytes with
    | Some bytes ->
        use stream = new MemoryStream(bytes)
        use player = new SoundPlayer(stream)
        player.PlaySync()
    | _ -> failwith "No bytes to sound."
```
    
With these four little functions, we are in position of emulating Shaqib's uber-cool sunglasses with this nice one-liner:

```fsharp
imageUrl |> analyzeImage analyzerKey |> translateText translatorToken "en" "es" |> toSpeech translatorToken "es" |> play  
```

Take an image URL, describe it, translate it, make it an speech, and play it. Almost natural language, eh?
You can get the source code for this functional little project [here](https://github.com/edgarsanchez/FunCognitiveServices), and as usual comments and suggestions are welcomed!