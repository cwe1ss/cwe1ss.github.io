---
title: 'Feature Folders: Controllers and Views'
date: 2014-07-02T00:00:00+01:00
categories:
- .NET
toc:
  enable: true
aliases:
- /post/2014/07/02/Feature-Folders-Controllers-and-Views/
---

The first step in our process to [a better folder structure for our MVC projects]({{< ref "introducing-the-asp.net-mvc-feature-folders-project-structure.md" >}}) is to make sure, MVC can resolve our Controllers and Views. This is our target structure:

* (Project Root)
  * Areas
    * (AreaName)
      * (FeatureName)
        * (FeatureName)Controller.cs
        * Index.cshtml
        * Edit.cshtml
      * ... (other features)
      * Shared
        * ... (area specific shared views like EditorTemplates, Layout-pages, ...)
    * ... (other areas)
    * Shared
      * ... (area independent shared views like EditorTemplates, Layout-pages, ...)
  * Features
    * (Feature2Name)
      * (Feature2Name)Controller.cs
      * Index.cshtml
      * Edit.cshtml
    * ... (other features)
    * Shared
      * ... (feature independent shared views like EditorTemplates, Layout-pages, ...)

Of course, if you don't want to use "areas" you only need the "Features" folder in your project. This also means, that if you move to this new structure, you can completely remove the old "Controllers" and "Views" folders.

<!--more-->

### Controllers

To support this structure for Controllers, you don't have to change anything in MVC since it does not force you to place them in a special folder! You can put Controllers into whatever folder you want. Resolving them is purely depended on your RouteConfig.

### Views

To support this structure for Views, you have to create a custom ViewEngine. As you can see in the following example, this can also be done very easily. Please note, that this code only supports *.cshtml-files. If you want to use *.vbhtml-files as well, you just have to duplicate the paths and change the extension to *.vbhtml.

```c#
public class FeatureFolderViewEngine : RazorViewEngine
{
    public FeatureFolderViewEngine()
    {
        // {0} ActionName
        // {1} ControllerName
        // {2} AreaName

        AreaViewLocationFormats = new[]
                                {
                                    "~/Areas/{2}/{1}/{0}.cshtml",
                                    "~/Areas/{2}/Shared/{0}.cshtml",
                                    "~/Areas/Shared/{0}.cshtml",
                                };

        AreaMasterLocationFormats = new[]
                                    {
                                        "~/Areas/{2}/{1}/{0}.cshtml",
                                        "~/Areas/{2}/Shared/{0}.cshtml",
                                        "~/Areas/Shared/{0}.cshtml",
                                    };

        AreaPartialViewLocationFormats = new[]
                                        {
                                            "~/Areas/{2}/{1}/{0}.cshtml",
                                            "~/Areas/{2}/Shared/{0}.cshtml",
                                            "~/Areas/Shared/{0}.cshtml",
                                        };

        ViewLocationFormats = new[]
                            {
                                "~/Features/{1}/{0}.cshtml",
                                "~/Features/Shared/{0}.cshtml",
                            };

        MasterLocationFormats = new[]
                                {
                                    "~/Features/{1}/{0}.cshtml",
                                    "~/Features/Shared/{0}.cshtml",
                                };

        PartialViewLocationFormats = new[]
                                    {
                                        "~/Features/{1}/{0}.cshtml",
                                        "~/Features/Shared/{0}.cshtml",
                                    };

        FileExtensions = new[]
                        {
                            "cshtml",
                        };
    }
}
```

Of course, if you use this new structure, you lose some of the built-in templating- and navigation-support in Visual Studio since VS does not recognize these folders as "Views"-folders. Therefore, the following things no longer work:

*   "Go To View" throws an error.
*   "Add View" adds the view to the old "Views"-folder.

Fortunately, ReSharper helps you with these issues since it contains built-in templates for views and also [supports our custom ViewEngine](http://blog.jetbrains.com/dotnet/2013/01/29/resharper-and-custom-aspnet-mvc-view-location/)!
