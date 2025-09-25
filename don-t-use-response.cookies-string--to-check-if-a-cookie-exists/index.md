# Don't use Response.Cookies[string] to check if a cookie exists!


### Update: Follow Up Posts

*   [Use wrappers to access your cookies, sessions, ...]({{< ref "use-wrappers-to-access-your-cookies-sessions.md" >}})
*   [Testable and reusable cookie access with ASP.Net MVC RC]({{< ref "testable-and-reusable-cookie-access-with-asp.net-mvc-rc.md" >}})

### The short explanation, if you don't like to read the entire story

If you use code like `if (Response.Cookies["mycookie"] != null) { … }`, ASP.Net automatically generates a new cookie with the name "mycookie" in the background and overwrites your old cookie! Always use the Request.Cookies-Collection to read cookies!

<!--more-->

### The long explanation and some useful advices at the end

You can access the Cookies-Collection in two different places in ASP.Net:

*   [*Request.Cookies*](http://msdn.microsoft.com/en-us/library/system.web.httprequest.cookies.aspx) gives you the cookies that are sent from the browser to your server.
*   With [*Response.Cookies*](http://msdn.microsoft.com/en-us/library/system.web.httpresponse.cookies.aspx), you can send cookies from your server to the browser.

### Storing cookies
To make your life easier (or harder, as you will see later), ASP.Net gives you the possibility to add a cookie to the browser this way:

```csharp
Response.Cookies["mycookie"].Value = "some value";
Response.Cookies["mycookie"].Expires = DateTime.Now.AddDays(10);
```

As you can see here, the .NET framework automatically generates the HttpCookie instance with the name "mycookie" in the background and adds it to the collection.

![HttpCookie.Get](/uploads/2009/HttpCookie-Get.png)

Sending cookies to the browser this way is perfectly fine.

### Reading Cookies

The important thing you have to know when reading cookies is, that the Response.Cookies collection is *empty* at the beginning of every request. The cookie with the name "mycookie" can only be found in the Request.Cookies-Collection!

If you use the following line to check the cookie, a new cookie with the name "mycookie" and an empty value gets added to the Response.Cookies Collection and *this overwrites your old cookie*! (See framework code above!)

```csharp
if (Response.Cookies["mycookie"] != null)
{
  // This automatically overwrites the existing cookie with an empty value!!!
}
```

The correct way to access cookies is by using the Request.Cookies-Collection:

```csharp
if (Request.Cookies["mycookie"] != null)
{
  // This is fine
}
```

### So remember the following rules

*   To read cookies, ALWAYS use the Request.Cookies-Collection
*   Only use the Response.Cookies-Collection to set or change cookies

### How can I make it better and more beautiful?

Accessing the Request.Cookies or Response.Cookies collection directly is lame! You can't easily test this code and you don't have all the other cool stuff like type safety and IntelliSense for the keys.

I will write a follow-up post right after this one with an detailed example on how you can do it better!

Thanks for reading,  
Christian Weiss

