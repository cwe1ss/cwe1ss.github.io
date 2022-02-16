---
title: Testable and reusable cookie access with ASP.Net MVC RC
date: 2009-01-28T18:35:00+01:00
categories:
- .NET
aliases:
  - /post/2009/01/28/Testable-and-reusable-cookie-access-with-ASPNet-MVC-RC/
  - /post/2009/01/28/Testable-and-reusable-cookie-access-with-ASPNet-MVC-RC.aspx/
  - /blog/post/2009/01/28/Testable-and-reusable-cookie-access-with-ASPNet-MVC-RC.aspx/
---

All good things come in threes, so I'm writing another post about how to access cookie or sessions. I got inspired by a comment from [Chris Marisic](http://www.marisic.net/), who suggested to use a more testable way for this stuff!

Previous posts about this topic:

*   [Don't use Response.Cookies[string] to check if a cookie exists!]({{< ref "don-t-use-response.cookies-string--to-check-if-a-cookie-exists.md" >}})
*   [Use wrappers to access your cookies, sessions, ...]({{< ref "use-wrappers-to-access-your-cookies-sessions.md" >}})

### Intro

Using static wrapper properties is a quick and easy way, but you can't unit test them because they access HttpContext.Current directly. This time I will show you, how you can build a fully unit testable and strongly typed way to access your cookies. As there has been Christmas time 2 days ago ([ASP.Net MVC RC1 was released](http://weblogs.asp.net/scottgu/archive/2009/01/27/asp-net-mvc-1-0-release-candidate-now-available.aspx) \*g\*) I’m using the latest MVC bits for my example!

### A Reusable Cookie Container

The cookie container is responsible for getting values out of and into the cookie collections. It does not know which concrete values I'm using in my application! This is implemented in a different level, so you can use this class for all of your applications!

In my implementation it's possible to store "objects" in cookies. I've implemented it this way because I don't want to convert all my DateTime, int, ... cookies every time! But I also don't want someone to save Lists or any other complex types, so my SetValue() method validates the type of the value and throws an exception, if it's not a value type or nullable value type. That's a little type checking, but I think it is worth it because cookies are set quite rarely!

<!--more-->

Here’s the interface:

```c#
public interface ICookieContainer
{
    bool Exists(string key);
        
    string GetValue(string key);
    T GetValue<T>(string key);
        
    void SetValue(string key, object value, DateTime expires);
}
```

I will just show the fundamental code here. If you want to see the whole implementation, please take a look at the code sample. (see bottom)

As you can see below, I've used the abstracted versions of HttpRequest and HttpResponse, which you get, if you use ASP.Net MVC. That's just one of thousand things I love about ASP.Net MVC. These classes can be used easily in unit tests. Notice that everything can be injected here. There’s no direct access to HttpContext.Current!

```c#
public class CookieContainer : ICookieContainer
{
    private readonly HttpRequestBase _request;
    private readonly HttpResponseBase _response;
    
    public CookieContainer(HttpRequestBase request, HttpResponseBase response)
    {
        // "Check" is a helper class, I've got from the "Kigg" project
        Check.IsNotNull(request, "request");
        Check.IsNotNull(response, "response");
        
        _request = request;
        _response = response;
    }
    
    public string GetValue(string key)
    {
        Check.IsNotEmpty(key, "key");
        
        HttpCookie cookie = _request.Cookies[key];
        return cookie != null ? cookie.Value : null;
    }
    
    public void SetValue(string key, object value, DateTime expires)
    {
        Check.IsNotEmpty(key, "key");
        
        string strValue = CheckAndConvertValue(value);
        
        HttpCookie cookie = new HttpCookie(key, strValue) {Expires = expires};
        _response.Cookies.Set(cookie);
    }
    
    // ... see code sample for full implementation
}
```

Here’s a sample unit tests that proves the testability of this code. I use [Moq](http://code.google.com/p/moq/) as my mocking framework.

```c#
public static class Mocks
{
    public static Mock&lt;HttpRequestBase&gt; HttpRequest()
    {
        var httpRequest = new Mock&lt;HttpRequestBase&gt;();
        httpRequest.Setup(x =&gt; x.Cookies).Returns(new HttpCookieCollection());
        return httpRequest;
    }
    
    public static Mock&lt;HttpResponseBase&gt; HttpResponse()
    {
        var httpResponse = new Mock&lt;HttpResponseBase&gt;();
        httpResponse.Setup(x =&gt; x.Cookies).Returns(new HttpCookieCollection());
        return httpResponse;
    }
}

// This method is from my CookieContainerTests class

[TestMethod]
public void SetValue_UpdatesExistingCookie()
{
    // Arrange
    const string cookieName = "myCookie";
    const string cookieValue = "myValue";
    DateTime cookieExpires = new DateTime(2009, 1, 1, 0, 0, 0);
    
    var httpRequest = Mocks.HttpRequest();
    var httpResponse = Mocks.HttpResponse();
    var cookieContainer = new CookieContainer(httpRequest.Object, httpResponse.Object);
    
    httpResponse.Object.Cookies.Add(new HttpCookie(cookieName, "oldValue"));
    
    // Act
    _cookieContainer.SetValue(cookieName, cookieValue, cookieExpires);

    // Assert
    HttpCookie cookie = httpResponse.Object.Cookies["myCookie"];
    Assert.IsNotNull(cookie);
    Assert.AreEqual(cookie.Name, cookieName);
    Assert.AreEqual(cookie.Value, cookieValue);
    Assert.AreEqual(cookie.Expires, cookieExpires);
}
```

That’s it! Now you have a testable and reusable cookie container!

### How to use it in your application

It's really easy to integrate this into your app! Just create an interface that defines all your application-specific properties you want to save in cookies and a concrete implementation of this interface that interacts with the cookie container.

```c#
public interface IAppCookies
{
    string UserEmail { get; set; }
    DateTime? LastVisit { get; set; }
}

public class AppCookies : IAppCookies
{
    private readonly ICookieContainer _cookieContainer;
    
    public AppCookies(ICookieContainer cookieContainer)
    {
        _cookieContainer = cookieContainer;
    }
    
    public string UserEmail
    {
        get { return _cookieContainer.GetValue("UserEmail"); }
        set { _cookieContainer.SetValue("UserEmail", value, DateTime.Now.AddDays(10)); }
    }
    
    public DateTime? LastVisit
    {
        get { return _cookieContainer.GetValue&lt;DateTime?&gt;("LastVisit"); }
        set { _cookieContainer.SetValue("LastVisit", value, DateTime.Now.AddDays(10)); }
    }
}
```

You can now inject this IAppCookies interface to your MVC Controller:

```c#
public class HomeController : Controller
{
    private readonly IAppCookies _cookies;

    public HomeController(IAppCookies cookies)
    {
        _cookies = cookies;
    }

    public ActionResult Index()
    {
        DateTime currentTime = DateTime.Now;

        IndexViewModel viewModel = new IndexViewModel
        {
            CurrentTime = currentTime,
            LastVisit = (_cookies.LastVisit ?? currentTime),
            UserEmail = _cookies.UserEmail
        };

        _cookies.LastVisit = currentTime;

        return View(viewModel);
    }

    public class IndexViewModel
    {
        public string UserEmail { get; set; }
        public DateTime LastVisit { get; set; }
        public DateTime CurrentTime { get; set; }
    }
}
```

Wow, you’re still reading :-)

That’s all I want to show here! If you want to see more about how the IOC is set up (I’m using [StructureMap](http://structuremap.sourceforge.net/Default.htm)) or anything else, take a look at the full code:

*   [Download full code sample](http://cid-16ce9c120fa181c9.skydrive.live.com/self.aspx/chwe.at%20blog/090129%7C_CookieContainerApp.zip)

I look forward to hearing your feedback on this!

Thanks for reading,  
Christian Weiss
