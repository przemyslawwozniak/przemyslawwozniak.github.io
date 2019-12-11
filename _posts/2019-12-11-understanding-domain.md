---
title:  "Understanding domain with Event Storming"
date:   2019-12-11 23:30:00 +0100
categories: programming
excerpt: Every experienced developer knows the rule "do not start with a code". How can one effectively get enough understanding of domain to translate it into code effortlessly? Use Event Storming.
---

I've recently started to work on a project from scratch, given only broad idea that came to my mind. Being experienced at *starting* new side projects (irony here), I developed my own process which is aimed at optimizing the path from idea to MVP. Just for the sake of keeping my morale high, I like to think about it as a "golden Graal of software development"... OK, maybe let's stop here :)

So, I'm starting something new, and have to take Grall off the shelf again. It's a little bit covered with dust, so I've decided I'll let you watch (but only partially) while I'm cleaning it.

It all starts with an **idea**. I like to let idea grow in my mind to become a **vision**. When I have a vision, I try to imagine myself the most vital **customer journey(s)** that take place in this project. But these are all sentences at the moment. We have to somehow distill from them the **domain knowledge**, which will be used as a basis for the initial core code - to be honest, the most important piece of code in the whole project, which will create product's value. To accomplish this goal, I use Event Storming - to be precise, it's initial phase, "understanding", understood as described in this [great article][https://spring.io/blog/2018/04/11/event-storming-and-spring-with-a-splash-of-ddd] published on Spring's blog by Polish developer Jakub Pilimon.

Below you can find my short note on "understanding", compiled from a few sources (including linked one above) and my own experience.

***

1. Domain events, e.g.:
  * order placed
  * invoice generated
  * new user account created
Domain events have to be important from the domain's experts point of view and written in past sentence.
Use orange stickers for domain events. Enhance them with yellow stickers, which brings additional information. E.g. "User enters a home page" + "User accepted cookies displayed with a modal".
Consider placing stickers from "first" event to "last" event in order to create timeline for events. At the end, verify them in backwards direction.
When domain events are written down, proceed to second step.

2. What has to be done for the domain event to happen? Commands, e.g.:
  * place order
  * generate invoice
  * create user account
Most of the time, commands will sound similar to domain events. Also, there is a high chance they'll transform into methods during the coding phase (but coding is the last thing to think about during event storming session, remember that).
Use blue stickers for commands sent directly to the system.
There is a variety of sources that trigger an event; not only commands, triggered by user, but also external systems, time (mark with a small sticker saying 'time') or another domain event (put another orange sticker next to triggered one).
Also, there is a special kind of 'commands', which are used to read model (i.e. query database) or display some view - these are called queries. Use green stickers for them.
What's more, you have to consider if there are any additional conditions which allow to perform the action? These conditions are called invariants. E.g. if the domain event is 'new user account created' and command is 'create user account', invariant could be 'rules for password are satisfied'. Invariants shall be represented as yellow stickers, put between domain event and command.
Commands written down? Proceed to third step.

3. Who triggers the command? Actors, e.g.:
  * invoice is generated when user hits 'Download invoice in PDF format' button
  * user account is created when user enters a link which confirms account creation, sent to his email
Use red stickers for actors.
All done? Now you understand your domain.

***

The result of understanding phase of Event Storming is a board tightly filled with stickers. It's like an unordered list of things to do. We are still missing a plan. To create it, follow "divide" phase described by Jakub. The first time you're actually touching a code is writing specifications based on what you've learned with Event Storming. It's also well-described in the linked article as a "implement" phase. I encourage you to read it.

Traditionally, Event Storming is done in "pen and paper" approach, because it's main idea is to be a common ground for domain experts and programmers. And hey, it's brain storming, so you shall not be distracted with tools. But, archiving the results could be tedious and for sure you'd like to do so for a future reference. Also, more and more teams are working remotely (there is a growing trend of "no office" companies, which I think is simply great), so some digital tools have emerged. You'll google them with no issues, but if you're a lonely wolf aka one man army, you can use [Lucidchart][https://www.lucidchart.com/blog/ddd-event-storming] for free.
