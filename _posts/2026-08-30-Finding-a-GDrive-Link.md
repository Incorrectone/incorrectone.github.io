---
title: Finding a GDrive Link
date: 2026-08-29 13:30:25 +/-1000
categories: [Web, Security]
tags: [security, webdev, auth, privacy]     # TAG names should always be lowercase
author: incor

# [true/false]

toc: true 
comments: false
math: false

pin: false

---

So there I was looking for the lecture slides for the midterm exams, and I noticed that the old college Google Drive link was replaced with a web app by the maintainer. I thought "meh..." a subpar webapp that might just be like a PDF reader. But no, I was wrong, and very wrong at that. It had a "fully functioning" login/signup page. 

## the Web App

To view the previously freely available drive documents, you had to create a compulsory account. That ticked me off a little, but since the mid terms were near, I hesitantly did. Now the bigger problem was the amount of information they collected for a site that was previously almost anonymous. It might just put Google to shame.

That was that. I went forward to look for the slides and I noticed a very familiar interface for accessing the documents, Google-Drive-like arrangement of files and folders and in fact the files were just links to Google Drive documents as well. So I figured that since they hadn't used any cloud service (like S3 buckets) but instead just put a wrapper in front of `googleapis.com` to present the files. Now that just made me want to just find and look for the root folder where they had put the files altogether.

## the snooping

I'd always figured the Google Drive API was simple enough that the drive folder URI would just be sitting in the Network tab in DevTools; find it, get the link, done. But that's not what was happening.

> Some of the text from here is speculation since after informing the developer, I did not pry anymore, and they took the site down.
{: .prompt-warning }

They apparently had made a Firebase database for the file structure (I think that's what they did.) That was being queried for finding the correct files, and / or it could be just the auth database (idk) but that does not matter.

The database was returning the links to the actual files; that was a win but not the end. I wanted the main drive link, not the individual files. Now, I don't do frontend. I don't know Firebase, I don't know how OAuth works, I code in C, so a person who knows how Firebase permissions and its APIs work could most likely have queried the database from the credentials I saw. But I did not know that. So I snooped further into the files that are actually loaded into the browser to do the basic stuff.

There I see the *Possessio carissima*, just hiding in plain sight. The authentication logic right on my browser's dev tools.

I want to emphasize how easy this file was to find. I did not have to run any tools, any descramblers, any exploits or anything, just by loading the site I could see it. The **hardcoded** list of the Admin Emails that had special privileges was just sitting on the top of the file.

## the admin page

Now I did not explore the whole file, because that was not the point and would have been very tedious. But the thought that comes to the mind is that *why* does that list even exist? In a normal frontend that would not even be needed. But this was not a normal frontend. This was college student's *magnum opus* to try to find new ways to seek rent.

I explored even further to find the use of that list. There it was. it basically told the web app to redirect the current user from a group of admin-only URLs if the user's registered mail was not in that list. There you could see the one, singular target URL path that was of any use to us. From the name you could see it was a directory of sorts. My brain started clicking and that directory could only mean one thing. All the files!

There I went to that URL path, only to get redirected back to the dashboard. Now this was solvable using a simple change that is available in all web dev tools. The "Override Script" tool which allows you to edit the script (and the list in it) in any way you want.

## the Data Leak

Now one could have started to explore that one specific URL, but there were many more pages that could have been important and dangerous if leaked. However to find the link I actually wanted; I first went to directory for that *Gazis Primis* root drive link.

But as always, exploring the rest is a must.

> There was PII in this section of the website. Nothing was taken, and it was reported to the developer immediately.
{: .prompt-danger }

But the spice (content, heh) must flow. While some of the stuff was configured correctly, like the analytics and the file list (on the surface). Most of the stuff showed error, showing that some of the authentication was indeed set up (correctly or not, idk). the part that actually mattered; the user data, was not.

More bonus information was also available if you had logged in from Google SSO with Gmail. There was more than just an email sitting in that record too. That at this point is not a list but a whole dataset. (The irony...)

## the speculation (and the LLMs)

> High on speculation. Also, yes I know buzzwords, buzzwords.
{: .prompt-warning }

Now one has to wonder wwhy this site was working this way? Why did the developer decide to make a hardcoded list of whitelisted emails? I think the answer is pretty simple and straightforward.

The site is made using free tools. There is nothing more to it than that. Cost savings. The developers most likely used free Firebase APIs, Drive APIs, and a free hosting platform. Meaning the site most likely does not have a 'backend' to speak of. All of the business logic is directly baked into the frontend. Now this would be fine if the site was made properly.

***It was not.*** The Devs most likely used LLMs to code most of this site (i.e. the frontend).

Here is my thought process: they most likely made the pages first, they worked fine but they did not check the user data section extensively (1). Then using dummy accounts they saw that the admin pages were visible to normal users as well, now that is a problem for a good looking website. So the LLMs / Agents were asked to allow access to only specific emails. Without thinking much the LLM just implemented the redirect mechanic (2). It passed on the visual testing (3), because "does it work when I click it" and "is it actually secure" are very different questions, and only one of them was asked. So it made it into prod (4).

## the aftermath

The aftermath was simple. I informed the developer of the API keys, the leaking information, and badly configured databases. The developer took the site down, I do not know how long it will take for them to actually make it safe (good thing I did not use the Google SSO).

> And the funny part is I found the Google Drive link regardless, from the directory page, soooo **BBBBIIINNGGGOO**.
{: .prompt-tip }

However, the things I explored are only part of the puzzle. The things I did only took  basic knowledge of web dev and very little time. I do not like web dev either, I am practically on the opposite end of the spectrum with programming in C, and C++. The database was, is, and most likely still is very misconfigured, and someone with more experience could have gotten all the data very easily.

The actual lesson may just be a simple one: do not trust unreviewed LLM code with your auth.