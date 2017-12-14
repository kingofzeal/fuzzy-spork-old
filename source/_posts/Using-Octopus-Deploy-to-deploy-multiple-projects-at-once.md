---
title: Using Octopus Deploy to deploy multiple projects at once
tags:
  - Octopus Deploy
  - Devops
  - Deployment
date: 2017-12-14 08:00:00
---


We use Octopus Deploy at my workplace to handle all of our releases. The tool is one of the best I've had the pleasure of dealing with, and the more I interact with it, the more I realize just how powerful, yet flexible, it is. I will gladly recommend it to anyone who is looking for a way of releasing any kind of software. 

However, there is always room for improvement. In this case, not necessarily improvement with the software package, but how we use it. <!-- more -->

## Environment Overview
Before we get to the specific problems, let me first go through what our deployment process generally looks like.

We operate on a two week release cycle. At the end of each cycle, we package all our products up and send it to Octopus. From there we create a release for each product, which is then pushed to a Staging environment for final testing. 

We have 4 major products that are released using 7 Octopus projects, and each of the products may or may not have changes during each release cycle. We offer our products as an on-premise package and as a hosted cloud-based SAAS offering, and have an active Beta program for customers who want early access to new features. To accommodate this, we have a total of 6 environments - `Development`, `Staging`, `On Site Beta`, `Hosted Beta`, `On Site Release` and `Hosted Release`.

Finally, because of our SAAS offering, we make heavy use of tenants - each of our customers has their own discrete copy of the software, but on a system which may be shared by other customers.

Overall, we likely have what many would consider a 'small' setup for Octopus - I have heard of companies that have 10s of environments and hundreds of projects. While we may be small, we are at the point where to take care of everything manually would take several hours of work on a frequent basis, not to mention be error-prone.

## So now what?
When all this is put together, there are a LOT of individual tasks to be run. It's relatively easy to queue everything all our projects and just let them all run, but if there are problems mid way through the process, it can be difficult to recover. This can be compounded by the fact that many of our products generally need to be deployed in a certain order, and while they do recover gracefully if they are done in the wrong order, it can result in some downtime until everything is complete.

One alternative that was discussed, which we did for a little while, was to do each project one at a time, and not starting the next until everything was deployed successfully. Since we aren't _quite_ ready for scheduling our deployments yet, this means one or more people need to stay up and baby sit the entire process, which can easily take a couple hours. If something did error out, then the time it took to identify and correct the issue would be added to the total time. This worked for a little while, but as we began to scale it was obvious this wouldn't work long term.

## Chain Deployments
Enter the Octopus step template `Chain Deployment`. This is a community-provided PowerShell script that makes use of Octopus' open API to allow one project to automatically trigger the deployment of another project. It is pretty configurable to allow for different applications, and it seemed like an obvious solution to our problem. 

Using this, we could have configured each of our projects to trigger the release of the next project in line, but since we also want to be able to deploy a specific product independently from the rest, we instead opted to make one more project, which we call `_Deploy`.

This project is basically only responsible for the deployment of all of our products at the end of the release cycle. The version number for each release of `_Deploy` matches that of our flagship product, and it's mostly just a series of Chain Deployment tasks. Each of the tasks are configured to wait until the chained deployment is complete before moving to the next, which effectively pauses the release process if an error occurs.

However, this was not as simple as creating a project with 7 Chain Deployment steps and calling it a day. As I mentioned earlier, all our products are released to a Beta program. In our case, we will release new code to our beta customers immediately after a release cycle before pushing it to our remaining customers after the following cycle. We can't just use Chain Deployment, as it would either always push the most recent version to all environments (or attempt to, anyway - the lifecycle would prohibit it), or we would have to edit the project steps every release to have the specific version number of each of the products.

That first option is unacceptable, and the second would run into issues if we have to release a patch or hotfix to a given product - we would have to create an entirely new release with the new version numbers and promote it along the lifecycle, which can take a while. In an ideal world, we could release only the impacted projects to the impacted environments, and then update `_Deploy` to use new version going forward.

## Making it all work
So how do we make it work then? Instead of the 7 steps you might expect, we have 14 - each of our Chain Deployment steps also has an associated Deploy a Package step to accompany it.

Keep in mind that the `_Deploy` project _still_ does not actually do any work - it only triggers the other projects to deploy. Instead, we scope the Deploy a Package steps to only work in a `Testing` environment (which coincidentally isn't used in any of our lifecycles), and then configure the Chain Deployment to reference the package version that is used in that step. Here's what it looks like:

![](Project Steps.png)
![](Deploy Step Details.png)

As you can see, we also group each Chain Deployment/Deploy a Package so we can skip the entire process and don't accidentally the whole thing.

The other nice thing of doing this is that when we create the release we get the same package selection list that every other product has, and it can be changed just like any other project mid-stream. This allows us to keep the process exactly the same from release to release, only changing the packages it uses to do so, which means we not only avoid the issue of the latest version going to an environment before we want, but we also can change the releases mid-cycle (for things like patches) by just updating the packages used.

![](Create Release.png)

A keen eye might notice that we only have 6 packages in the list. This is because our flagship product isn't quite done with the conversion to Octopus yet, so there is no package to deploy. Instead, we use the version number of the `_Deploy` release to determine what version number to use for that project.

I also feel I should point out that this only works because all our projects are configured to have release numbers that match a specific package's version, and that package is the one referenced by the `_Deploy` project. 

## Now what?
We've been using this project for a little while now, and we've found that setting this up not only made things easier, but more efficient. Since the output of the chained deployments (or at least, some of it) is displayed when looking at the `_Deploy` process, it's easier to gauge how well things are progressing. Not having every possible task queued all at once also helps the overall process cope with changes or occasional one-off tasks that need to be done during a deployment. It's also easier to on-board new people to how the deployment process works - create the release and click a button. 

No matter how we look at it, this has been a net gain for our deployment process, and I hope it inspires you to think outside the box to do something simple yet powerful for yours.