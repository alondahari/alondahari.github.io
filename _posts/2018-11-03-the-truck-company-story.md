---
layout: post
title: The Truck Company Story
image: /img/truck.png
---
The CEO of my company likes to use analogies to make a point, so I thought it would be fitting for me to dedicate this post to her,
and follow one analogy to try and explain some coding related principals to "non tech savvy" types.

In this analogy, we'll follow Bob and Willie, two competitors in the trucking business. In a nutshell, Bob is doing everything wrong and Willie is killing it when it comes to following best practices.

Hopefully this would help explain why we dedicate some of our time as developers to perfect our code, even when sometimes it seems like there's no tangible benefit to it.

_I never worked in the trucking industry, so I'm totally making all of this up..._

**Follow a style guide**

As much as it is important to have code that works well and performs well, it's also important to have code that's _readable_. We have a lot of tools to help us with this as developers (linters* of different kinds), but it's the easiest principal to throw out the window when you're pressed to deliver fast. In which case, you end up with a bunch of unmaintainable spaghetti code.

The main reason we need our code to be readable, is maintainability. Every time we write some code, we have to consider the fact that in the future someone else might need to read our code and understand in a reasonable amount of time what we did. Our future selves also fall in the category of "someone else" (when revisiting code I wrote 6 months ago, it might as well be someone else that wrote it).

--

Let's get to our guys. They're writing a set of instructions to follow for delivery of the same route (this is before google maps):

___Bob's version:___

You head west on greenwood past a school and after the bridge you take a right on wall after 2 miles you take a left on 8 and you pass the library and right after you keep going straight and then you take a left on Broadway you'll see the place on your right after 1.5 miles it's called "Donny's" you can't miss it.

___Willie's version:___

- Head west on Greenwood
- Right on wall
- Continue 2 miles on Wall
- Left on 8th
- Left on Broadway
- Continue 1.5 miles on Broadway
- "Donny's" on your right

Now, both instructions would work, and in the case of coding, it doesn't really matter what's easier to read, because a machine is "reading" it (assuming it won't take a toll on performance). The question is: _if a mistake was made, and you need to take a right on Broadway instead of a left, which set of instructions would be easier and faster to fix? which one of those you'd feel more comfortable fixing and knowing it's now "good to go"?_

**The DRY principal**

According to the DRY (don't repeat yourself) principal, yo should try to strive and re-use chunks of code. It is the opposite of WET (write everything twice). It helps the maintainability of our code in two ways: it makes our code base shorter and easier to understand, and allows making changes at one place instead of many places in case we need to.

Our boys are writing down an day's itinerary for their drivers. Bob already learned to write in bullet points...

___Bob's version:___

- Drive to Virtucon, sign in, unload the merchandise, go through the manifest with the client, sign off on the count.
- Drive to Globex, sign in, unload the merchandise, go through the manifest with the client, sign off on the count.
- Drive to Oscorp, sign in, unload the merchandise, go through the manifest with the client, sign off on the count.
- Drive to Nakatomi Trading Corp., sign in, unload the merchandise, go through the manifest with the client, sign off on the count.


___Willie's version:___

- Deliver to Virtucon, Globex, Oscorp and Nakatomi Trading Corp.
- At each location: sign in, unload the merchandise, go through the manifest with the client, and sign off on the count.

Now, they both find out that the drivers forget to put on the hazard lights and they need to add that to the instructions. _Which set of instructions is easier to update? where would it be easier to miss a spot?_

On top of that, Willie's instructions resemble more closely how our brains work. Think about it, after the first delivery the driver probably remembers what he needs to do. It's also easier for them to see that they need to do exactly the same thing at each location, and therefore it's more _readable_ (and thus easier and faster to maintain).

**Testing**

*Linters are tools that analyze our code and, following a set of rules, alert us if something looks wrong, much like a spell checker. For example, I can tell my linter to alert me any times a line of code is more than 80 characters long, or I type a comma instead of a dot.
