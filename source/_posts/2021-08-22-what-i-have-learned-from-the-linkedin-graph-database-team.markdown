---
layout: post
title: "What I Have Learned From the LinkedIn Graph Database Team"
date: 2021-08-22 19:48:33 -0700
comments: true
categories: 
---

I had worked for the LinkedIn graph database team for 5+ years and we successfully built a [graph database](https://engineering.linkedin.com/blog/2020/liquid-the-soul-of-a-new-graph-database-part-1) serving the entire LinkedIn economic graph. In this post, I want to share what I have learned. Disclaimer: many of the words and wisdom are from my great colleagues.

<!-- more -->

### All incidents are gifts
Incidents are opportunities for us to fix bugs, improve the stability of the system and improve the process of handling incidents. We should treat them as gifts and learn as much as possible out of them.

### All incidents should be novel
This basically means that we should never make the same mistake twice. Once an incident happens, we try as hard as we can to fix it and make sure it will never happen again with the same root cause.

### Hardware failure is common at scale
We have, on average, 2 DIMM failures per week so we should design our software in a fault tolerant way.

### API is sticky
Once clients start to use the exposed APIs, they become extremely sticky. That means we need to design them carefully since changing them afterwards is [costly](https://www.joelonsoftware.com/2004/06/13/how-microsoft-lost-the-api-war/).

### Logging is talking to the user/operator
How programs talk to humans has a huge impact on the rate at which mistakes can be fixed. If programs tell humans exactly what is wrong, that rate can be very fast. If programs are silent or overwhelm humans with too much information, that rate can be extremely slow. When there are too many spurious errors, people get alerts fatigue and they will overlook the real problems.

When we write logs in our code, we need to remember that the audience is not just us but also people that may not be familiar with the entire codebase like SREs. That means the log messages should be crystal clear and actionable. Imagining how frustrating it is when oncalls get paged at 2am and they have no clue what those log messages mean and how to act on them. [Here](https://spark.apache.org/error-message-guidelines.html) is a guideline of how to write good error messages.

### Comment on why
Code comments should say why the code is there instead of what the code does. What the code does should be clear from the code itself. If it is not the case, then we should refactor the code to make it clear instead of adding a comment. A large decaying comment is frequently just an apology for crappy code. Don't accept the apology. Fix the code. Don't give up until you try your best. Then, as a last resort, write the comment.

### Good enough is not enough
The math is simple: 0.8 * 0.8 * 0.8 * ..... = 0. If every time we just achieve good enough, then eventually it will become zero/failure. We should never settle and always try to do as best as we can.

### Conduct code review
Code review is not just about finding bugs. We should also think about how we can rewrite the code in a better way that is easily understandable and unlikely to cause future bugs.

Also reading other people's good reviews allows us to learn not only from our own mistakes but also the mistakes of others.

### Write design document
"Writing is nature’s way of letting you know how sloppy your thinking is" by Leslie Lamport. The very act of writing the design document helps to clarify the design itself. It also helps people to learn or understand the system in the future.

### [Keep your eyes open](http://blog.jjyao.me/blog/2021/08/18/keep-your-eyes-open/)
