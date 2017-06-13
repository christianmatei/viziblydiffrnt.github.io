---
layout: post
title: "Taking Tableau Further: Create Your own Content Management Tool, Part 1"
date: 2017-06-13
tags: [Tableau Server, APIs, Content Management]
header:
  teaser: /assets/images/content_lifecycle.jpg
---

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/content_lifecycle.jpg">
</p>

Consider the lifecycle above as it relates to your Tableau content. You work hard to Plan, Develop, and Test your content and then you make it available to your audience by publishing it to Tableau Server. After your users sign off on it, it goes into Production where (hopefully) they use it to extract insights that impact their business. 

But what happens next? How do you know if it's being used? What if the question your visualization was built to answer is no longer relevant and your workbooks are going unused? How do you manage your content at scale?

With recent releases of Tableau Server, Tableau has expanded their Enterprise capabilities to tackle many of these questions. Content analytics provide a window into who is using your content and how frequently. Data driven alerts notify users of changes in metric values or when extracts fail to update. But what happens when people stop using your visualizations? Wouldn't it be great if Tableau could intelligently detect content that's no longer being used and manage it for you? Today there is no out-of-the-box way to do this effectively and at scale.

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/content analytics.png">
</p>

*Content Analytics on Tableau Server*

Your organization may have a retention policy or guidelines for scheduling content but today all of those rules need to be implemented manually by your admin team. And if you run a lean team that doesn't have the bandwidth to regularly clean up 1000's of workbooks the task can be daunting. Don't despair, there is hope.

Using Tableau's APIs (official and unofficial) we can create our own Content Management Tool that will allow us to meet our requirements, however unique they might be.


## Determining Your Feature Set

Before we start coding it's really important to know what features you want your Content Management tool to handle. This helps you focus on an outcome and avoid chasing after things that might only have marginal value. Through my client work I've seen a variety of content management challenges so I chose to focus here on ones that are applicable to a wide range of Tableau environments. For my solution, I chose the following:

1) Check for all workbooks and datasources that have not been used in N days (configurable based on need). This will form the basis for the actions we'll take further on.

2) Be able to "Whitelist" content that should not be altered. This could be because it has significant operational need or because you know the C-Suite looks at it or because it took a really long time to build and you want everyone to know that. Just kidding about that last one. Any reason to preserve content will do. 

3) Based on periods of activity, or inactivity, the tool should decide what course of action to take. Potential actions include:

* Disable the Extract Schedule
* Remove the Workbook/Datasource from Server
* Reschedule the Extract to a More Efficient Time Slot

4) If a particular piece of content is to be removed, send the Content Owner an email explaining why their content is being removed and include their content as an attachment so it's not lost forever. The attachment should be a .twb or .tds file to minimize the file size and the email should include a thumbnail image of the workbook if one is available.

5) Log all actions and produce a summary of each action taken for review, **aka Demo Mode**.

## Putting Ideas into Action

Now that we know what features to build out, we need to figure out how to get the information we need. For this feature set we'll need:

* Access to [Tableau Server's Postgres](http://onlinehelp.tableau.com/current/server/en-us/data_dictionary.html) database - so we can review content usage patterns and get scheduling details.
* The [Tableau Server REST API](http://onlinehelp.tableau.com/current/api/rest_api/en-us/help.htm#REST/rest_api_ref.htm#API_Reference%3FTocPath%3DAPI%2520Reference%7C_____0) - for getting our whitelisted content, downloading workbooks and datasources, deleting content from the server.
* [The Tableau Vizportal API](https://viziblydiffrnt.github.io/blog/2017/01/26/documenting-tableau-vizportal-api) - for disabling extracts, changing extract schedules
* Access to an SMTP Server for sending emails to users. Alternatively, you could subtitute email notifications for Slack or whatever your company uses for collaboration.

I chose to code my solution with Python but you could use any language that supports connections to Postgres and HTTP requests for working with Tableau's APIs. Another thing to consider is who your audience is going to be. Are you designing for an experienced IT admin that knows how to modify a config file or could your end user be semi-technical analytics manager that would prefer a sleek web-app? These questions can impact your design decisions so think about how you want users to interact with your tool before you begin coding.

In the next part of this series I'll dive into what data points we need to retrieve to enable our features and the design methodology I used to build this content management tool. 