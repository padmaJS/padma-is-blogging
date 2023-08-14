---
layout: post
title:  "Navigating UI Design Iterations"
background: '/img/2023_8_14/dp.jpg'
subtitle: "Sharing my journey on how I finalized my page design"
---

## Navigating UI Design Iterations: From Flux to Focus

### Introduction:
Select an UI design is dynamic and iterative process. Multiple changes and revisions before coming to a decision. I went through a process of selecting a soothing design for a site in my **elixir liveview** project where I had an **Assignment** details page listing tests related to the assignment. I really did not have much of design ideas as well because I am more of a backend person. Anyways! I am here to share what I did and went through to finalize on my design.

### What I had in mind
  I had imagined that I wanted all the details relating to the assignment to be shown in the same page so the user won't have to leave the site at all. I wanted to do this because of my previous design. 

My schemas are like this.
```
A Module has many Assignments. Assignment belongs to a Module.
An Assignment has many Tests. A Test belongs to an Assignment.
A Test has many TestSupportFile. A TestSupportFile belongs to a Test
```
 Where for a user to reach or view assignment test support file. He had to go through modules -> assignment -> test -> test_support_file. But I wanted to cut it down to modules -> assignment. Where the **Assignment** had all details necessary. I also thought of a few designs. Firstly an **accordion**. I really liked the way accordion show cases data and hides if user clicked off.
 
 But ofcourse I ran into problems. Firstly I wasn't sure how I was going to map the all **test** data into a single accordion. Bit by bit I managed to create somewhat working accordion UI. But I wasn't really happy about it. What I had was Spaghetti code. Too complex for even me. If something broke I wasn't confident to fix it.

Then I also went though implementing same thing with a **table**. But same problem. Became too complex. 


### How I finalized the design
I was going through my mails and saw the design of the dashboard. Click onto the dashboard options and the other side of screen changes. I thought it could work perfectly.

So I sat infront of my pc and started implementing it. I took a grid, listed all the **Test**  as button on left side and through event handling, listed the **Test** details on the other side. As easy as that. I really loved how simple and fewer lines of code I required to do that. On top of that, data management was easier, code wasn't complex anymore because the code was something I was very familiar with. Just like that I added test details also and I had final design that I loved.

##### Assignment Details page
![Screenshot 1](/padma-is-blogging/img/2023_8_14/ss1.jpg){: width="100%" }
##### When I click on a test
![Screenshot 2](/padma-is-blogging/img/2023_8_14/ss2.jpg){: width="100%" }
I went with a really minimalist design.


### Conclusion
I had envisioned to implement a UI where all details of a schema was accessible but did not know how. I went through process of finding designs and iterating over a the ideas. I had a initial design in my mind which I managed to make somewhat working but was complicated. But with an open mind and patience I found what I loved. Being flexible is key. If I has stuck to just one idea I would not have better ways of solving the problem I faced. Changes help to explore more. And also just be patient.