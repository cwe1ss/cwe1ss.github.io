---
layout: post
title: Use wrappers to access your cookies, sessions, ...
date: 2009-01-26 18:45
redirect_from: 
  - /post/2009/01/26/Use-wrappers-to-access-your-cookies-sessions-e280a6/
  - /post/2009/01/26/Use-wrappers-to-access-your-cookies-sessions-e280a6.aspx/
  - /blog/post/2009/01/26/Use-wrappers-to-access-your-cookies-sessions-e280a6.aspx/
---

As described in my [previous post]({% post_url 2009-01-26-don-t-use-response.cookies-string--to-check-if-a-cookie-exists %}), I will give you some more details about how you can access your cookies in a type-safe and easy way!

<!--more-->

### Update: Read the follow up post
*   [Testable and reusable cookie access with ASP.Net MVC RC]({% post_url 2009-01-28-testable-and-reusable-cookie-access-with-asp.net-mvc-rc %})

The simplest way to do this is by using a little wrapper class like this one:

{% highlight c# %}

    using System;
    using System.Globalization;
    using System.Web;
     
    namespace CookieWrapper.Web
    {
        public static class MyCookies
        {
            public static string UserEmail
            {
                get { return GetValue("UserEmail"); }
                set { SetValue("UserEmail", value, DateTime.Now.AddDays(10)); }
            }
     
            public static DateTime? LastVisit
            {
                get
                {
                    string strDate = GetValue("LastVisit");
                    if (String.IsNullOrEmpty(strDate))
                        return null;
                    return DateTime.Parse(strDate, CultureInfo.InvariantCulture);
                }
                set
                {
                    string strDate = value.HasValue ? value.Value.ToString(CultureInfo.InvariantCulture) : null;
                    SetValue("LastVisit", strDate, DateTime.Now.AddDays(10));
                }
            }
     
            private static string GetValue(string key)
            {
                HttpCookie cookie = HttpContext.Current.Request.Cookies[key];
                if (cookie != null)
                    return cookie.Value;
                return null;
            }
     
            private static void SetValue(string key, string value, DateTime expires)
            {
                HttpContext.Current.Response.Cookies[key].Value = value;
                HttpContext.Current.Response.Cookies[key].Expires = expires;
            }
        }
    }

{% endhighlight %}

All you have to do is create a static property for every cookie that you would like to work with. As you can see you also have the Expires-times administrated in one single place!

Now you can access the values as seen below and you don't have to worry about the cookie implementation-details in every place.

{% highlight c# %}

    tbLastVisit.Text = MyCookies.UserEmail;
    MyCookies.LastVisit = DateTime.Now;

{% endhighlight %}

Of course, you can also use this same approach for working with session data or any other key-based collection.

Thanks for reading,  
Christian Weiss
