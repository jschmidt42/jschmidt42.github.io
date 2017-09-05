---
layout: post
title: Building or not a platform
category: thinking
tags: [platform, framework, application]
---

<a href="http://xkcd.com/974/"><img src="http://imgs.xkcd.com/comics/the_general_problem.png" alt="The General Problem" /></a> <br/>

Recently I am struggling with this simple question: <b>Should we develop a generic platform</b>? To put more context, should we develop a generic platform to build a specific application first? Let's say you are doing an Next-Gen animation system, should you invest an important amount of time and energy to build that platform that could solve all type of system? My opinion on that matter is that you should do the less possible on the platform side and focus on the application you want to deliver. I am not saying that you shouldn't try to make a good platform or framework, I am just saying that if you try to develop that platform at the same time, there is good chance you'll have a lot of problem in the long run. Its like those game company that try to make a game engine and a game both at the same time from day one, most of them fails to do so. It is not much different for application frameworks.

In example, Unreal Engine wasn't made to create Unreal, Unreal was created and out of this success story, they then made a more useful engine that they were able to deploy to other teams and companies.

I really think, that you should focus on doing the best application you can, and then, when you know your application is good enough, invest some time and effort to extract the good stuff to build your platform slowly. Is it not why refactoring exists?

Anyway, don't get me wrong, I am not saying a framework is not important, and that you should go and hack your system from day one, I am saying that its best to show that your application works before than having a framework that pretends to solve everything generically but fails to solve your main goal.

People who prefer to have a sexy framework tend to write an complex API before writing code that will eventually call these API calls. What a error this is! Because most of them, are the type of person who would kill an ant with a Bazooka! They generally over-engineered their interface with templates, abstract names that means nothing, etc. Also, it is important to note that more your code is generic, more slowly it runs in the general case! The bad performance of these frameworks is most of the time detected to late in the development and really hard to solve.

So here my short list of rules I tend to follow in my software development cycles:

1. Always focus on the application, not the framework -> People buys applications, not platforms (ok SAP is the exception!)
2. Write client code before service code -> code that never gets called is useless
3. Only implement stuff that gets used -> when you over-engineer you tend to make stuff that is hard to debug and also stuff that don't get used from the start tend to be more bogus 
4. Design things that are easy to debug -> code easy to debug, make your development velocity much better on the long run, keep template and complex class hierarchy to a minimum
5. Fix your bugs when they are found -> otherwise you are looking for bigger problems (<a href="http://msinilo.pl/blog/?p=820">read broken window theory</a>)
6. Make sure everyone on your team are on the same page -> Cohesion, convergence and complicity are the keys to productivity!

Good luck

