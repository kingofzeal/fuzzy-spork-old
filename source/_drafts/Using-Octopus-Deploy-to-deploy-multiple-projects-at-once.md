---
title: Using Octopus Deploy to deploy multiple projects at once
tags:
  - Octopus Deploy
  - Devops
---

We use Octopus Deploy at my workplace to handle all of our releases. The tool is one of the best I've had the pleasure of dealing with, and the more I interact with it, the more I realize just how powerful, yet flexible, it is. I will gladly recommend it to anyone who is looking for a way of releasing any kind of software. One thing we're currently working on is not just deploying all our products, but doing so as a 'one-button' deployment.

Let me first go through some background on how we handle deployments.

We operate on a two week release cycle. At the end of each cycle, we package all our products up and send it to Octopus. From there, each discrete product has an associated release, which is then pushed to a Staging environment for final testing. We have four major products that are released with seven Octopus projects, and each of the projects may or may not have any changes from a given release cycle. Additionally, we make heavy use of tenants as majority of our customers have their own copies of each software package in a cloud hosted environment. Finally, we have a total of four production environments, `On Site Beta`, `Hosted Beta`, `On Site Release` and `Hosted Release`.

### What does all this mean?
Basically, when we go to actually release everything there are a LOT of individual tasks to be run. It's relatively easy to queue everything up and just let it run, but if there are problems mid way through the process, it can be difficult to recover. This can be compounded by the fact that many of our products generally need to be deployed in a certain order, and while they do recover gracefully if they are done in the wrong order, it can result in downtime until everything is complete.

One alternative that was dicussed was to do each project one at a time, and not starting the next until everyting was deployed sucessfully. Since we aren't _quite_ there for scheduling our deployments yet, this means one or more people need to stay up and baby sit the entire process, which can easily take a couple hours. If something did error out, then the time it took to identify and correct the issue would be added to the total time. This worked for a little while, but as we began to scale it was obvious this wouldn't work long term.

### Chain Deployments
Enter the Octopus step template `Chain Deployment`. This is a community-provided PowerShell script that makes use of Octopus' open API to allow one project to automatically trigger the deployment of another project. It is pretty configurable to allow for different usages, and it seemed like a no-brainer to use this to solve our problem. Using this, we could have configured each of our projects to trigger the release of the next project in line, but since we also want to be able to deploy a specific product independantly from the rest, we instead opted to make one more project, which we call `_Deploy`.

This project is basically only responsible for the 'total' deployment of all of our products at the end of the release cycle. The version number for each release of `_Deploy` matches that of our flagship product, and it's mostly a series of Chain Deployment tasks. Each of the tasks are configured to wait until the chained deployment is complete before moving to the next, which allows us to effectively pause the entire release if an error occurs.

However, this was not as simple as creating a project with seven Chain Deployment steps and calling it a day. As you may have noticed from our environments earlier, we release our products via a Beta process. In our case, we will release new code to betas immediately after a release cycle, and let it site there during the next cycle before pushing it to our main customers. We can't just use Chain Deployment, as it would either always push the most recent version to all enviroments (or attempt to, anyway), or we would have to edit the project every release to have the specific version numbers of each of the other projects.

That first option is unacceptable, and the second would run into issues if we have to release a patch or hotfix to a given product - we would have to update the process with the new version numbers and promote it along the lifecycle, which can take a while. In an ideal world, we could release only the project in question to the impacted environments, and then update `_Deploy` to have the new version going forward.

### Making it all work
How do we make it all work? Well, instead of seven steps we have fourteen - seven for Chain Deployment and seven for Deploy a Package. 

Keep in mind that the project _still_ does not actually do any work - it only triggers the other projects to deploy. Instead, we scope the Deploy a Package steps to only work in a Testing environment (which coincedently isn't used in our lifecycle), and then configure the Chain Deployment to reference the package version that is used in that step. Here's what it looks like:

<image placeholder>

We also group each Chain Deployment/Deploy a Package so we can skip both at once if we need to.

The other nice thing of doing this is that when we create the release we get the same package selection list that every other product has, and it can be changed just like any other project mid-stream. This allows us to keep the process exactly the same from release to release, only changing the packages it uses to do so, which means we not only avoid the issue of the latest version going to an environment before we want, but we also can change the releases mid-cycle (for things like patches) by just updating the packages used.

Keep in mind this only works if all the projects are configured to have release numbers that match a specific package's version, and that package is the one referenced by the `_Deploy` project. 