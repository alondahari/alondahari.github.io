---
layout: post
title: The Truck Company Story
author: alon
categories:
  - Coding Principals
---

The CEO of my company likes to use analogies to make a point, so I thought it would be fitting for me to dedicate this post to her,
and follow one analogy to try and explain some coding related principals to "non tech savvy" types.

In this analogy, we'll follow Bob and Willie, two competitors in the trucking business. In a nutshell, Bob is doing everything wrong and Willie is killing it when it comes to following best practices.

Hopefully this would help explain why we dedicate some of our time as developers to perfect our code, even when sometimes it seems like there's no tangible benefit to it.

_Disclaimer: I never worked in the trucking industry, so I'm totally making all of this up..._

## Follow a style guide

As much as it is important to have code that works well and performs well, it's also important to have code that's _readable_. We have a lot of tools to help us with this as developers (linters\*\* of different kinds), but it's the easiest principal to throw out the window when you're pressed to deliver fast. In which case, you end up with a bunch of unmaintainable [spaghetti code](https://en.wikipedia.org/wiki/Spaghetti_code).

The main reason we need our code to be readable, is maintainability. Every time we write some code, we have to consider the fact that in the future someone else might need to read our code and understand in a reasonable amount of time what we did. Our future selves also fall in the category of "someone else" (when revisiting code I wrote 6 months ago, it might as well be someone else that wrote it).

Let's get to our guys. They're writing a set of instructions to follow for delivery of the same route (this is before google maps):

**_Bob's version:_**

You head west on greenwood past a school and after the bridge you take a right on wall after 2 miles you take a left on 8 and you pass the library and right after you keep going straight and then you take a left on Broadway you'll see the place on your right after 1.5 miles it's called "Donny's" you can't miss it.

**_Willie's version:_**

- Head west on Greenwood
- Right on wall
- Continue 2 miles on Wall
- Left on 8th
- Left on Broadway
- Continue 1.5 miles on Broadway
- "Donny's" on your right

Now, both instructions would work, and in the case of coding, it doesn't really matter what's easier to read, because a machine is "reading" it (assuming it won't take a toll on performance). The question is: _if a mistake was made, and you need to take a right on Broadway instead of a left, which set of instructions would be easier and faster to fix? which one of those you'd feel more comfortable fixing and knowing it's now "good to go"?_

## The DRY principal

According to the DRY (don't repeat yourself) principal, yo should try to strive and re-use chunks of code. It is the opposite of WET (write everything twice). It helps the maintainability of our code in two ways: it makes our code base shorter and easier to understand, and allows making changes at one place instead of many places in case we need to.

Our boys are writing down an day's itinerary for their drivers. Bob already learned to write in bullet points...

**_Bob's version:_**

- Drive to Virtucon, sign in, unload the merchandise, go through the manifest with the client, sign off on the count.
- Drive to Globex, sign in, unload the merchandise, go through the manifest with the client, sign off on the count.
- Drive to Oscorp, sign in, unload the merchandise, go through the manifest with the client, sign off on the count.
- Drive to Nakatomi Trading Corp., sign in, unload the merchandise, go through the manifest with the client, sign off on the count.

**_Willie's version:_**

- Deliver to Virtucon, Globex, Oscorp and Nakatomi Trading Corp.
- At each location: sign in, unload the merchandise, go through the manifest with the client, and sign off on the count.

Now, they both find out that the drivers forget to put on the hazard lights and they need to add that to the instructions. _Which set of instructions is easier to update? where would it be easier to miss a spot?_

On top of that, Willie's instructions resemble more closely how our brains work. Think about it, after the first delivery the driver probably remembers what he needs to do. It's also easier for them to see that they need to do exactly the same thing at each location, and therefore it's more _readable_ (and thus easier and faster to maintain).

## Testing

Testing your code is high in cost and high in rewards. The cost is time and effort (usually not the most exciting work either). The problem is that the reward is often invisible - it only shines through when things _don't_ go wrong. When they do save the day though (usually only visible to the developers), it sure feels good!

Willie has a habit: every morning he goes out and checks the oil and water levels on all of his trucks, before they go out for the day. He also demands that all of his drivers walk around the truck before every time they fire it up, to make sure there isn't anything alarming going on. Once a month, he has his mechanic check all of his trucks and give them some TLC.

Bob doesn't do any of that. As a result, he and his drivers get the job done a little faster, and he actually makes a marginally higher profit! Not to mention, he saves up a bunch of money, not having a mechanic on hand.

You know what's gonna happen next though, right? One day, one of his trucks breaks down on the highway because it was slowly leaking oil for months and nobody caught that! Now Bob has to fix the truck, lose money on the deliveries, and overall look like the amateur that he is!

---

I'd say those are the big three, and a lot more principals in coding would just fall within those categories. Hope this helps bridging some communication gaps between the geeks and the jocks!

\*\*Linters are tools that analyze our code and, following a set of rules, alert us if something looks wrong, much like a spell checker. For example, I can tell my linter to alert me any times a line of code is more than 80 characters long, or I type a comma instead of a dot.
