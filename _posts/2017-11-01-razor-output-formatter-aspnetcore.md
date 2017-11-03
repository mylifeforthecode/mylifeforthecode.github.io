---
layout: post
status: publish
published: true
title: Using Razor With Asp.Net Core Api Content Negotiation and FormatFilter
author: Shawn Rakowski
author_login: srakowski@live.com
author_email: srakowski@live.com
---
_Published: 01-NOV-2017_

I've been working with Asp.Net Core lately to make API calls from a Vue.js SPA. When returning an object result I've enjoyed how Asp.Net Core MVC automatically negotiates JSON as the response format. Recently, however, I've desired to hit the same endpoint to get back the same data but optionally in a nicely formatted HTML doc, assuming I send the appropriate Accept:text/html header that is. Technically, I could have conditionally returned an ObjectResult __XOR__ a `View()`, but HTML should be just another content format for an API, and there is no reason the Controller should have to concern itself with it. Essentially, I wanted Asp.Net Core MVC to return JSON by default when `return Ok(model);` or just `return model;` except when I have sent the Accept:text/html header __AND__ have created a Razor view to pump the Model through.

To accomplish this I needed to look at how Asp.Net Core does content negotiation. There are a few articles on this and the Asp.Net Core docs [give an overview](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/formatting). By default, Asp.Net Core returns JSON for every ObjectResult except where additional formatters are present and the content has been negotiated. Looking around the web, [another great Shawn of the world has provided a blog post on the same topic.](https://wildermuth.com/2016/03/16/Content_Negotiation_in_ASP_NET_Core) These examples, while providing a good overview, only really show the Xml formatter that is provided by Microsoft via Nuget for XML. This is great if Xml is what you want, but I to create an HTML doc via Razor I had to dig deeper.

In the same Microsoft docs half of the answer is provided. To run our ObjectResult through some custom code we need a [Custom Formatter](https://docs.microsoft.com/en-us/aspnet/core/mvc/advanced/custom-formatters). This allows us to override some methods and convert some object to our desired output. They give an example of converting some contact model to a vcard that describes most of what you need to know on this front. Great, half way there!

This is where the slog of putting together a bunch of disparate things hit me, and why I decided to record it in a blog post in case I ever needed it again.

The next step was to figure out how I can convert some object into a rendered Razor view. This involved figuring out how to invoke the Razor Engine manually to render the object. Luckily this answer was provided by some searching and I landed on this article by ppolyzos: [https://ppolyzos.com/2016/09/09/asp-net-core-render-view-to-string/](https://ppolyzos.com/2016/09/09/asp-net-core-render-view-to-string/). Which gives a good exmaple how to use the IRazorViewEngine (injected by Asp.Net Core DI) to render a Razor view to a string. Aha! This is the other half I needed!

While I said that Custom Formatters and rendering Razor Views to a string are each 1/2 the solution they are more like 45% each of the solution. There is a remaining 10% I had to put together.

Registering output formatters in Asp.Net Core Startup.cs looks something like this (from the Vcard example):
```
services.AddMvc(options =>
{
    options.OutputFormatters.Insert(0, new VcardOutputFormatter());
};
```
Because we have to instantiate and insert the output formatter we lose our Dependency Injection capabilities. So how do we get at our IRazorViewEngine?

To my luck the answer was right there in the documentation...
```
You can't do constructor dependency injection in a formatter class. For example, you can't get a logger by adding a logger parameter to the constructor. To access services, you have to use the context object that gets passed in to your methods. A code example below shows how to do this.
```

Essentially, you have access to the services via the HttpContext, the example they provide retrieves the logger:
```
IServiceProvider serviceProvider = context.HttpContext.RequestServices;
var logger = serviceProvider.GetService(typeof(ILogger<VcardOutputFormatter>)) as ILogger;
```

Huzzah! Now we have everything we need to put this together!

## Example

Lets say we have a simple database of Gargoyles (my fav 90's cartoon), a super simple dummy active record may look something like this:

{% gist 2a47cf32c53f3320512b64cce208ec57 Gargoyle.cs %}
<!-- <a href="https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-gargoyle-cs">https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-gargoyle-cs</a> -->

And you have a controller that serves up the complete list of Gargoyles or an individual Gargoyle by name:

https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-gargoylescontroller-cs

Running the above model and controller through a base Asp.Net Core MVC project you get output like this:

For `/api/gargoyles` you get:

```[{"name":"Goliath","color":"RebeccaPurple","isLeader":true},{"name":"Hudson","color":"Brown","isLeader":false},{"name":"Lexington","color":"Olive","isLeader":false},{"name":"Broadway","color":"Cyan","isLeader":false},{"name":"Brooklyn","color":"FireBrick","isLeader":false},{"name":"Bronx","color":"MediumSlateBlue","isLeader":false}]```

For `/api/gargoyles/lexington` you get:

```{"name":"Lexington","color":"Olive","isLeader":false}```

Now what if I want these to models (the enumerable and the individual Gargoyle) to be returned as an HTML doc? Using [P[Postman](https://www.getpostman.com/) or some other tool I can modify the Accept header to request the `text/html` mime time, but Asp.Net, knowing nothing about this, will return JSON. So we can create a couple of Razor views, one for an individual Gargoyle:

https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-gargoyle-cshtml

and one for an enumerable of Gargoyles:

https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-gargoyles-cshtml

However, MVC has no tie into these without using the `View()` method on the controller, and we don't want to have to put some conditional logic in our controller to handle this. This is where the custom formatter comes in:

https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-razoroutputformatter-cs

This combines elements from the Custom Formatter code and the example of how to render a Razor view to a string. Now all we need to do is modify the AddMvc options in our Startup.cs to register our formatter and we're ready to go!:

https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-startup-cs

_Note: I use `Add(value)` vs. `Insert(0, value)` (as the custom formatter shows) because I want JSON to take precedence_.

In the example I am explicitly providing a way for the RazorOutputFormatter to resolve the name of a View. This could be done via convention based on the Type name or possibly by using Attributes + Reflection on the models you would pass through this (like custom serializer attributes). I am also assuming that I can write any type out throug this method. __The OutputFormatter class does provide overrideable methods to control what types or requests this formatter supports__. Look up the `CanWriteType` and `CanWriteResult` methods for more information on how to do this.

Now, a request through postman that looks like this:

```
GET /api/gargoyles/lexington HTTP/1.1
Host: localhost:58357
Accept: text/html
Cache-Control: no-cache
Postman-Token: 8b97ee35-5b86-09b7-8b8a-f83abca9a032
```

(_Note the Accept header_)

Will return Html like this:

```
<h2>Lexington</h2>
<hr />
<h3>About</h3>
<p>
    This gargoyle is  
    <span>not</span>  the leader of the Gargoyles.
</p>
<p>
    This gargoyle's color is:
    
    <div style="width:100px; height:100px; background-color: Olive"></div>
</p>
```

And every other, non `text/html` gives us back our JSON. Great this is exactly what I wanted!

## FormatFilter

One thing I discovered with this is that it's a pain to use Postman to get html that I cannot easily inspect. What it would be nice to do is open up my browser and get back HTML when I want to view HTML and JSON when I make a normal request. To do this I discovered that Asp.Net core provides a way to pass the negotiatied format type as part of the route. This requires a couple of attribute changes to our controller:

https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-gargoylescontroller-ff-cs

`[HttpGet("~/api/{_:regex(^(gargoyles)$)}.{format?}")]` __WAT?__

In the documentation for FormatFilter included in the [Formatting Response Data documentation](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/formatting) they provide a nice example of getting a single object after an id. What they don't tell you is how to use this on the base Get() route and that .{format?} must come after a route parameter. If you attempt to do something like `[Route("api/[controller].{format?}")]` you get some exceptions. After some Googling + trial & error I landed on this hacky mess where you provide a default route with a constraint, ugly but it works! To show I did some due diligence... I found this GitHub Issue on the topic with no meaningful resolution: [https://github.com/aspnet/Mvc/issues/3553](https://github.com/aspnet/Mvc/issues/3553).

One final thing we have to do is to tell MVC what Mime type the .html extension should resolve to. This is also done when configuring MVC in the Startup.cs file:

https://gist.github.com/srakowski/2a47cf32c53f3320512b64cce208ec57#file-startup-ff-cs

Huzzah!

Now we can provide the .HTML extension to our request and get back nice HTML. If we do not provide an extension, or if we provide the .JSON extension then we get JSON. Perfect!

You can see a demonstration of the behavior in this video:

https://youtu.be/v2bsZ7LSR10

## Thank you!

That's all for this post, thanks for checking it out!

You can find a full example of this code on GitHub here: [https://github.com/srakowski/RazorOutputFormatterExample](https://github.com/srakowski/RazorOutputFormatterExample)

Cheers!