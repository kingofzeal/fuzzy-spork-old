---
title: Managing Expectations
comments: true
date: 2017-08-02 13:33:27
tags:
---


This morning I realized that, as programmers, we frequently make assumptions about the software we write which seem to be reasonable, but aren't completely thought out.<!-- more -->

I have an older (second generation) iPad that I mostly use for playing games and managing a media server I have set up in my house. Occasionally I will use it as a note-taking device, but honestly it can't do much more, and I don't need it to. Since it's an older device, it's no longer receiving the latest-and-greatest iOS, and seems to be going through a kind of planned obsolescence that many other people have also noticed in aging iOS devices.

I recently downloaded a new game for it, an idle-builder type game. As with most games of this type, it has a mechanism where you can continue to earn the game currency even when the app is closed. However, the (in this case, problematic) implementation gives some insight into the mind of the developer. I obviously don't have access to the code, and I can only guess at what's going on, but the theory I have points to the developer having reasonable but incorrect assumptions that lead to a way of processing that is efficient (and if slightly changed could even be pretty clever), but equally incorrect.

Much of this is exposed because my hardware is older, slower and generally less stable than most of the devices on the market. As such, things tend to crash. A lot. I've gotten used to it, and generally don't tolerate apps that frequently crash before even getting to a point where I can interact with them. However, in this case it's just stable enough that I can play for a decent amount of time before it gets to that point. What I noticed was that when recovering from a crash, I would get the offline compensation not for the time it took me to reopen the app, but since the last time _I gracefully closed it_. 

I found that doing certain actions can trigger a crash fairly reliably (again, on my device - I don't suspect this is possible on newer devices). As I played with it some more, I found that when I perform normal actions that increase how much I get while offline and trigger a crash, it comes back online and gives the reward based on the _current_ (now updated) rate, not the original rate. Combine this with a continuously growing time frame and I was able to advance much quicker than was probably intended by the developer.

## So what's going on?
First, I want to point out that I'm not an iOS developer, and have never touched iOS languages, so I may not be using the correct terminology. From what I can tell, the app is wired into an event that is fired whenever the app goes into standby. This event is raised if you go to the app switcher, when a notification box is displayed, etc - basically any time the app loses focus for any reason. When the event is raised, a local DateTime setting is set with the current clock value. Then, when the app is restored from the standby state, it takes `(DateTime.Now - [Saved DateTime]) * [current offline rate]` and gives you that amount of currency.

At the face of it, this seems a perfectly reasonable calculation. Since the DateTime is saved whenever the app goes into _any_ standby state, including if you open the app switcher to manually kill the app, and the offline rate doesn't change while you aren't in the app, and it makes it difficult to "game" the system. This is still prone to adjustments of the system clock (ie, manually adjusting the clock forwards or backwards to make it seem as if more time has passed than it actually has), but those can be countered by other means and I didn't check to see if they were handled or not.

However, it seems the saved DateTime value is _not_ updated on a crash, or any other kind of event for that matter (like buying something, or literally any user interaction) - only when the app goes to standby. As a result, I'm able to exploit the instability of the app running on my device to force a crash, and then reload it, and take advantage of the fact that the app doesn't know that it's been opened before. Additionally, since all the other saved values of the game (like the offline earning rate) are saved as soon as possible, when the calculation is performed at start up, you not only get credit for the time but at the higher rate.

## Real world applications
I write this not from the perspective of how to take advantage of a programmer's fallacy, but what we as programmers can learn from this. And for me, the lesson is to challenge all your assumptions. The biggest assumption this developer made was that the event that fires on standby will _always_ fire, no matter what. There are other minor assumptions that were made, including that because a a device has a compatible OS version the app can be run on that device, but when combined, they allow someone to take advantage of the built-in mechanics to do things that weren't necessarily intended.

In this case, it's just a game - an argument can be made that nobody is really being harmed. However, the same lesson can be applied to other, perhaps more critical, areas as well like:

  * "We don't need HTTPS/SSL because it's an internal resource", or "it's secured by a VPN"
  * "We don't need to validate information on the server because the client can handle that and having it in two places isn't DRY"
  * "The operating system didn't change any relevant APIs, so we can support the older version to be more compatible"
  * "We have control over the API and the client code, so we can change anything to suit our needs"

Some assumptions might be valid, and in some cases the cost to remedy a faulty assumption might be deemed too high for the risk it exposes. The goal is not to make sure everything is perfect, but to examine the often unspoken reasons for doing things a particular way, make sure it's clear _why_ it was done, and think of uncommon events that could lead to unexpected behavior.