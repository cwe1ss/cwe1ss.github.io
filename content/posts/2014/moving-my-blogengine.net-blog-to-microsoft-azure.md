---
title: Moving my BlogEngine.NET blog to Microsoft Azure
date: 2014-03-29T00:00:00+01:00
toc:
  enable: true
aliases:
  - /post/2014/03/29/Moving-my-BlogEngineNET-blog-to-Microsoft-Azure/
---

I thought, after 5 years since my last post it might be time to write a new one! But that's not so easy! When you login to the backend after such a long time, everything feels just wrong: The design is ugly, the blog engine is outdated, I'm not sure if my hosting provider still is the best choice for me and I don't like my domain anymore! So, instead of just writing a post about something, I decided to change all of these things first and tell you about this process!

<!--more-->

### Get a running BlogEngine.NET instance within minutes

I was always interested in <strike>Windows Azure</strike>&nbsp;[Microsoft Azure](http://blogs.msdn.com/b/windowsazure/archive/2014/03/25/upcoming-name-change-for-windows-azure.aspx) but never really had the time to do something useful with it. Therefore I decided to take this opportunity and move my blog to it. I already had access to the Azure Management Portal, so I didn't have to go through some registration process. Since I also wanted to update my blog to the latest version of BlogEngine.NET I decided to start with a new installation and migrate my data afterwards (moving data from 6 posts is not so hard :-) ). Luckily, with Azure's Gallery, it's extremely easy to setup an instance of BlogEngine.Net. I just followed these instructions: <http://blogs.msdn.com/b/webdev/archive/2012/10/12/blogengine-net-and-windows-azure-web-sites.aspx>

After that, I just had to go through a quick export/import process to move my data from my old blog to the new one. I started with a "free" web site but quickly scaled it up to a "shared" one because I wanted the site to be running on my custom domain.

### Deployment with Git and the App\_Data folder

The default template of BlogEngine.NET 2.9 is quite nice, however I wanted to make some minor changes. Sounds like a good opportunity to test another great feature of Microsoft Azure: Git deployment.

[According to this tutorial](http://www.windowsazure.com/en-us/documentation/articles/web-sites-publish-source-control/), I setup a Git repository for my web site within the Azure portal and cloned the repository to my notebook. Again, this was very straightforward and worked immediately. This also has the advantage that you now have a backup of your remote files on your home network.

But there's one important thing to take care of when you use Git deployment with BlogEngine.NET. BlogEngine.NET by default stores data in files within the App\_Data folder. This is quite nice since you don't have to pay for a SQL Server database! However, if you use Git deployment it overwrites this folder, since it's now under source control. Therefore I added the "App\_Data" folder to the .gitignore file. Unfortunately, the deployment still did overwrite changes. I guess this happened, because on the server, the App\_Data folder was still a part of the git folder.

To overcome this, I tried a different way: First, I copied my local App\_Data folder to some backup directory on my notebook. Then I removed the .gitignore file and really deleted the App\_Data folder from my local and remote repository (so if you do this as well, please note that you will have a downtime!). After that, I manually copied the App\_Data folder back to Azure with FTP. As a last step, I re-created the .gitignore file with the App\_Data-exclusion and also moved the App\_Data folder back on my machine.

As a result, the "App\_Data" folder is no longer monitored by Git and will not be touched when Azure does a deployment. Of course, whenever you need a up-to-date version of your App\_Data folder on your PC for development purposes, you have to manually download it from Azure.

### Some warnings about this

*   I'm pretty sure this is not the best way to handle this. As far as I know, you shouldn't store user generated content on your server but instead use an Azure Storage account for it. Having user files on your server has several disadvantages: you can't scale out your servers, you don't have any replication or backups and so on.
*   I'm not sure if the current deployment behavior, which does not touch untracked folders will stay that way forever. I wouldn't be surprised if they do a real sync someday because actually, the git repository should be 100% consistent with the web folder.

This means, *I do not recommend this for anything else than completely uncritical things*! If you want to do this, make sure you regularly backup your App\_Data folder by e.g. doing a scheduled FTP download every day.

For this blog, I'm fine with those risks for now but if I happen to need a storage account anyway or if I will blog more in the future I will definitely move the data to a storage account.