---
title: On deployments and distribution
date: 2016-10-11 07:00:00
tags: 
- devops
- build
- deployment
---

Ever since I started programming, I've been fascinated with what you can do with a codebase - not just the act of programming, but building and deploying it as well. For a programming company, the less time programmers have to work on building or deploying their code, the more productive they can be. The automation of these things can have a huge benefit.

I'm sure many who may be reading this already have something in place to automatically build code, either on check-in, at a specific time, or both. Some may even have a system in place to automatically deploy code internally or to customers. But what is the real benefit from these things?

In many ways, having automated build and deployment solutions are as invaluable as have a version control solution. It doesn't ultimately matter what you use to accomplish the task, but having something puts you in a much better place than not having anything. I, and I'm sure many others, have gotten so used to having such systems that I run my own to take care of things. Even this simple blog is version controlled and automatically deployed. At this point, it's more habit than anything. My personal projects are all run through a build process on commit (I haven't gotten to a deployment, for various reasons, but I'm working on it). If there is a perceived benefit from doing this on projects/programs that have little to no impact, how much more benefit is there for a company whose existance relies on getting code built and out the door?

## Automated Build Solution
This is the easy one. If you can't build your code, then you have nothing to ship. But why should you have an automated build solution? Simply put, reliability. There's an interesting mentality that arises from using source control: Code that isn't checked in doesn't exist. Having a build environment separate from your development environment ensures that the only code that can be built is the code that is checked in. Sure, you can (and will) miss check-ins that cause a build to fail, but that's the point. Even more dangerously, without a separate machine, it's possible (and even inevitable) that you will build a feature that wasn't meant to be released.

With a dedicated machine to build code, you can avoid all these issues. Additionally, with the right set of triggers and configuration, you won't even need to give up your own time. Simply download the last build's artifacts and you're ready to deploy. This is even more helpful if your development cycle and deployment schedule don't exactly line up - simply pull the last build from the last day of your development cycle. 

Building on a dedicated machine also helps with stability. There are plenty of stories of projects that will, for one reason or another, build on one machine but not another. By eliminating the variance between machines (by using the _same_ machine), you can accurately say 99% of the time that the problem is with the code, not the environment.

It's also beneficial to have a build solution in place for a historical purpose: most solutions provide an easy way to track published artifacts over time. If a version of your code is released that has a bug, it's easy to rollback to the last known working version, since you generally still have those artifacts. Without them, the only thing that can be done is to attempt to fix the bug, often while facing immense pressure from executives and customers. 

## Automated Deployment Solution
There's nothing that says the build solution and the deployment solution can't be the same thing. I've worked with a build environment that has a large number of deployment steps, and a single build step. There's nothing wrong with such a system (and in some cases it can make sense), but know that it might not always be the best option.

Oddly enough, build solutions are generally found in most programming businesses but deployment solutions aren't as common. Many times, the software being released doesn't have a well-defined customer base, or a customer base that is so broad that such a system is deemed impractical. However, even in those situations a deployment solution could be a reasonable time saver.

If a product is being deployed solely for internal use, a deployment of a product takes an abnormally long time, or if there are many complicated steps to deploy a product, you will almost always benefit from an automated deployment system. Not only does it help to organize where things should be deployed to, but it can do everything that is needed accurately and reliably. This isn't to say that there won't be problems - deployments need testing, just like everything else. However, you can be assured that every deployment, given the same settings, will always do the same thing. Once it has been set up, you will generally not need to look at it again.

## Other Thoughts
Personally, I have always used [TeamCity](https://www.jetbrains.com/teamcity/) for my build system, even for personal projects. It has an easy-to-use interface, allows for running arbitrary scripts (helpful if you want to make it a deployment system), gets updates on a regular basis, and has a lot of plugins and features to support any type of project. The best part: it's free for up to 20 configurations and 3 build agents (runners).

For a deployment solution, I've used TeamCity to handle that as well, and it does a reasonable job at it. It requires more configuration than probably should be expected, but it still performs as expected. However, for more complex deployments, I've grown fond of [Octopus Deploy](https://octopus.com/). It isolates deployment processes from deployment targets in a way that makes it very easy to expand either. The focus is on the Microsoft stack (with nice native support for IIS and Windows Services), but can be used for almost anything. It is also available for free (for up to 20 of any combination of projects, target machines and users), but the time it takes to set it up and configure it doesn't lend itself well to personal projects, as they are generally only deployed to a single location. 

Finally, don't discount other systems, such as [Travis CI](https://travis-ci.org/), which excel at projects that are intrepreted, like Javascript and PHP (like this blog, which is generated with a Node package `Hexo`). Since these aren't compiled in the traditional sense, such systems can be used to run unit tests and deploy on success. Travis in particular has great integration with Github, where you can see Pass/Fail statuses and even enforce checks on pull requests.

 
I'm sure many who are reading this are already using most, if not all, these things, but for those who still are not: Take the time now and look into setting something up. You'll thank yourself for it tomorrow.