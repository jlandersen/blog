+++
author = "Jeppe Lund Andersen"
categories = ["Visual Studio", "Visual Studio Team Services", "ALM", ".NET", "Build"]
date = 2016-01-19T19:09:34Z
description = ""
draft = false
slug = "ten-subtle-visual-studio-team-services-features"
tags = ["Visual Studio", "Visual Studio Team Services", "ALM", ".NET", "Build"]
title = "Ten Useful Visual Studio Team Services Features You Might Not Know"

+++

In this post I have gathered ten useful features of Visual Studio Team Services. Many of these are not new features, but you had to dig a little to find out they were there.  In the recent months the UI has received several improvements and some of these are now more visible, without having to dive deep in the collection administration. There are also a couple of recent additions that should make your life easier!

## 1. Multiple Code Repositories in a Single Project
When you create a project you get a Git or TFVC repository in the same process - but it does not end there. You can have additional repositories in the same project. Simply go to the Code section, and in the upper left corner click the repository and choose "New Repository" or "Manage Repositories". 
![Creating additional repositories](/images/2016/01/Screen-Shot-2016-01-16-at-15-49-31.png)

Bonus info - until recently, any additional repository had to be of the same type as the default one, but it is now possible to have both Git and TFVC repositories in the same project!

## 2. Edit Files Directly in VSTS

Speaking of code repositories. Not only can we browse the files in repositories, we can also make changes and commit them directly in VSTS! Select a file in the repository explorer to see its content and click the *Edit* button. You can now edit the file, enter a commit message and hit save, and the file is now updated.

![Edit files directly in Visual Studio Team Services](/images/2016/01/edit.png)

## 3. Pin It to the Dashboard 
The dashboard recently received a major overhaul including an improved wizard and configuration experience for managing dashboard widgets. But you can also quickly spice up the dashboard with some work item query you just created or build history from a build definition. Spread throughout the service, the *Add to dashboard* menu item is available on some elements.

![Dashboard pin shortcuts](/images/2016/01/shortcuts.png)

## 4. Make Your Manager Happy - Work in Excel!
I have yet to meet a project manager that does not live half of his/her  professional life in Excel. But no worries, you can still make a strong case for VSTS in your development projects - everything related to work items can be handled directly in Excel, exactly how it has always been with TFS. Go to "Team" in the ribbon and hit "New List". You will be prompted to connect to a VSTS account. Once connected, select a project and choose a query that retrieves the work items you wish to work with.

![Connect Visual Studio Team Services to Excel](/images/2016/01/excelscreen1-1.PNG)



![Work with Visual Studio Team Services directly in Excel](/images/2016/01/excelscreen3.PNG)


## 5. Speed Up Build With Parallel Builds
Life is short, so there is no reason to wait unnecessary time for build artifacts. If you need to produce different types of artifacts, e.g. for different platforms, you can speed it up by letting VSTS schedule builds of each configuration in parallel if possible. Go to your definition options and use the *MultiConfiguration* and tick the *Parallel* setting. Enter a build variable to create parallel builds based on, and use comma separation for the variable value when queuing a build. If you have multiple available build agents that meets the demands of the definition, they can now work on each artifact in parallel.

![Parallel builds in Visual Studio Team Services](/images/2016/01/Screen-Shot-2016-01-19-at-18-07-06.png)

![Queueing a parallel build in Visual Studio Team Services](/images/2016/01/Screen-Shot-2016-01-19-at-18-00-39.png)

![Parallel builds in action](/images/2016/01/Screen-Shot-2016-01-19-at-18-03-43.png)


## 6. Fast Switch Between Projects
If you have multiple projects going, and find the need to switch between them, you can quickly switch between these. See that project name in the left side of the top menu bar that acts as a breadcrumb? It seems that many are unaware that this is in fact a project menu. Click it and get direct access to recently accessed projects and more.

![Fast switch between projects in Visual Studio Team Services](/images/2016/01/Screen-Shot-2016-01-18-at-20-39-13.png)

For even faster switching - use the keyboard shortcut P to open the menu, and use the keyboard arrows to navigate the items. 

## 7. Keyboard Shortcuts
Bringing us to the next point - keyboard shortcuts! A recent and welcoming addition - check out all of the shortcuts in the [official documentation](https://msdn.microsoft.com/en-us/library/vs/alm/overview/reference/keyboard-shortcuts).

![Keyboard shortcuts for Visual Studio Team Services](/images/2016/01/Screen-Shot-2016-01-18-at-20-42-31.png)

## 8. Personal Alerts
A project can have alerts set up by an administrator, but you can always configure personal alerts for a project if you want to be notified whenever something of value happens. Simply navigate to the project, click your profile in 
the upper right corner and choose *My alerts*.
![Accessing personal alerts i Visual Studio Team Services](/images/2016/01/Screen-Shot-2016-01-19-at-18-09-58.png)

![Creating personal alerts in Visual Studio Team Services](/images/2016/01/Screen-Shot-2016-01-19-at-18-08-51.png)

## 9. README's for Project Repositories
In #1 I described how a single project can have many code repositories. But it may also be an idea to include a description of these for other team members. Include a README.md file in the repository and it will appear under *Welcome* in the Home hub of your project. VSTS supports general markdown and GitHub-Flavoured markdown.

![Making readme's for repositories in Visual Studio Team Services](/images/2016/01/Screen-Shot-2016-01-19-at-18-14-09.png)

Bonus info - you can create and edit these directly in VSTS similar to editing files in #2.

## 10. Save Your Customized Processes
It used to be that you could only choose between three locked process templates, which was very different from the heavy customizable process template system in TFS. However, much work has happened in this area, and we can now create new templates that modify the existing ones with e.g. new fields. Navigate to the collection administration page and choose the *Process* tab. Here you can make new process templates and save these for later use when creating new projects.

![Customized processes in  Visual Studio Team Services](/images/2016/01/testets.png)

Bonus info - it was recently announced at the Connect 2015 virtual conference that Visual Studio Team Services will have feature parity with TFS on process customization in the future!