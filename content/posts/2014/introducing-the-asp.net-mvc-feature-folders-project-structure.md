---
title: Introducing the ASP.NET MVC “Feature Folders” Project Structure
date: 2014-04-07T00:00:00+01:00
categories:
- .NET
toc:
  enable: true
aliases:
  - /post/2014/04/07/ASPNET-MVC-Feature-Folders-Project-Structure/
---

### What’s the problem with the default ASP.NET MVC folder structure?

Which of these requirements is more common?

*   Change something in every view, controller or model of your project
*   Add a new field to your model X, show this field to the user, make it editable, the value must be validated against some fancy rules, ...

I guess we are on the same page if we see the second one as more common. I would go as far as to say that if you have the first requirement you're either working on a major relaunch or you're not using layout pages, css, abstract classes, [insert random reusability gadget here] correctly.

<!--more-->

By default, the ASP.NET MVC project structure advices you to keep every *concept* in its own area – you therefore might end up with a structure like this:

* Content
  * CustomerImages
    * AnIconSpecialToCustomers.png
  * Customers.css
* Controllers
  * CustomersController.cs
* Models
  * Customer.cs
* Repositories
  * CustomerRepository.cs
* Scripts
  * Customers.js
* Views
  * Customers
    * Create.cshtml
    * Index.cshtml
* ViewModels
  * Customers
    * IndexViewModel.cs
    * CreateViewModel.cs

As soon as you have more than 3 controllers, this becomes hard to navigate. ASP.NET MVC's solution for having a better structure is to use "Areas", however in my opinion they do not solve the problem I'm talking about. To complete the second requirement I've mentioned, you still have to navigate through many folders, because most probably, you don't have a distinct views-guy, a distinct model-guy, a distinct controller-guy, ... in your company. It's a lot more common that e.g. only one or two people are working on all of these mentioned files.

### Grouping files by feature

When I'm talking about a feature, I understand it as a sum of files that are needed to create a user benefit. Therefore, with structuring files by feature, the project structure could look like this:

* Customers  
  * Images  
    * AnIconSpecialToCustomers.png
  * Create.cshtml
  * CreateViewModel.cs
  * Customer.cs
  * CustomerRepository.cs
  * Customers.css
  * Customers.js
  * CustomersController.cs
  * Index.cshtml
  * IndexViewModel.cs

Think again of our second requirement and of some of the advantages with this structure:

*   You immediately get an overview about how the feature might be implemented.
*   You immediately see which files *might* be affected by the requirement. You don’t have to check every concept folder to see if there even is a corresponding file. (there might not be a js-file for every feature, ...)
*   Every affected file is in one folder. The required navigation in the Solution Explorer is kept to a minimum.
*   In your source control system, you can look at the entire change history of this feature on one folder.
*   If you have to implement a new similar feature, you can copy this one folder and use it as a starting point.
*   ...

### Why is there a M in ASP.NET MVC?

I would like to make an exception of my previous structure: It's important to understand that the ASP.NET MVC framework itself (System.Web.Mvc) does NOT give you any built-in support for "models". If you require persistent data, you are allowed to use whatever technology you want (Entity Framework, NHibernate, raw ADO.NET, ...). Yes, the project templates by default already reference Entity Framework, but again, this is a separate library and ASP.NET MVC has no dependency on it.

In my opinion this is a very good thing! The traditional three-tier architecture (data, business logic, presentation) still is one of the most important concepts for structuring software systems. ASP.NET MVC clearly targets the presentation tier and shouldn't cover responsibilities from other tiers.

For this reason, we have to move the files "Customer.cs" and "CustomerRepository.cs" into a separate library. However, everything else in our folder belongs to the presentation layer.

### What’s next?

I plan to do follow-up posts that address the challenges and also possible solutions for this structure, so stay tuned!