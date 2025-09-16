## From failing to cracking to passing GSoC 2025
In the [previous blog](https://dev.to/thesynthax/my-google-summer-of-code-journey-part-1-573f), I shared my story of dreaming, attempting, and failing GSoC. This one is about learning from my **earlier mistakes**, applying my knowledge, getting **accepted** into GSoC, and eventually passing it. I will discuss the details and technicalities in Part 3 of this blog.

## Before getting accepted (Oct 2024 - Apr 2025)
It is October 2024, and I’m in my **pre-final year**. Over the summer, I had built some projects (more about them in the upcoming blogs), and I’d decided I wanted to dive into **lower-level areas of software development**: operating systems, **kernels**, networking, virtualization, and **graphics**. I started learning about system calls, kernel development, Linux internals, drivers, etc., but couldn’t go very deep because of time and college work.

By the end of this semester (**November**), I had three organizations in my mind: The **FreeBSD** Project (OS), Haiku (OS), and FFmpeg (graphics). Now there is a reason for this. These orgs have a good track record of returning to GSoC each year, and their project ideas are posted on the ideas page all year round.

Now,

1. Haiku was written in C++, and the community wasn’t active.
2. **FFmpeg** was tempting (filters, **encoders**, Vulkan, SIMD), but my college senior, who did GSoC 2024 with them, **strictly warned me against it** due to poor documentation and community support.
3. That left **FreeBSD**, which I eventually picked by January 2025. (Lucky choice, since Haiku didn’t even make it to GSoC 2025.)

Now things were different this time. I chose an organization that is **not crowded**, unlike most webdev organizations, like Rocket.Chat, so I did not have to worry about much **competition**. This is an **advantage** of choosing a tech stack like this (C, OS, kernels, etc).

I went through the idea list and chose two projects to make proposals for:

1. **mac_do(4) and mdo(1) improvements** (kernel security module)
2. **adding compressed qcow2 images creation using mkimg(1)** (userland tool)

This time, I **didn’t contribute** at all (which was a **risky** move), but I immediately started **contacting mentors** once orgs were announced. I read the FreeBSD Handbook, studied some code, and pestered ChatGPT endlessly, constantly learning new things about the things required to work on the project. I **got replies** from **both mentors**, answering all my queries.

By March 2025, I was writing proposals for both projects **simultaneously**. It took me **23** days. I sent drafts to both mentors; one replied (kernel project) with some corrections, but said it **looked solid**. I was more confident about the **userland** project; it felt easier than kernel-level work. Unlike the previous year, this time I **focused well on the proposals**. The mentor and I exchanged emails for a few days before I submitted my final proposal on April 8, 2025.

## Getting Accepted and Community Bonding (May 2025)
May 8, 2025, it’s 11:30 pm. No email.
11:35 pm. Still nothing.
11:42 pm. I got an email. I got **rejected**.

![GSoC 2025 Acceptance email](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y7wtaohrw8esztq0mjfo.png)

I was devastated; those countless hours of learning, understanding deep concepts, and creating proposals, gone. I was shocked because I was pretty confident, but then, as I was accepting my reality that I wasn’t cut out for GSoC, I got an email saying, “GSoC 2025: **Congratulations**, your proposal with The **FreeBSD** Project has been accepted!”. (The previous mail was for the other proposal that I submitted.)

I was ecstatic! A dream of **seven years** is finally coming true! I was hoping that the other project got selected, but at this point, it didn’t matter. I got selected for the “**mac_do(4) and mdo(1) improvements**” project.

During the community bonding period, nothing much happened because my mentor was away for work, and I was chilling too, not putting much effort into learning more. This was a **mistake** that came back to bite me during the **coding period**.

## The Coding Period (Jun 2025 - Aug 2025)
### June
My mentor and I connect on Signal. We have our first call to discuss our first task on the `mac_do(4)` kernel security module (the most difficult of the three tasks). And **I was blank**. What I wrote in my proposal only covered **half of what was required**. When the remaining details were being discussed, I didn’t know what to do or how to do it.

Anyway, I started **coding**. After about three weeks, I opened a PR and took a break for a few days, while my mentor **reviewed** it, and I came to know that whatever I had coded was **complete and utter BS** (well, not exactly, but…). I had to **refactor** my code from scratch, **rewrite** most of the functions, and manage **memory** well while handling the **kernel-level concurrency** of the main `struct conf` that I had to work on. This task was difficult due to these concepts. Debugging was brutal; every memory management mistake risked a kernel panic and VM restart. I refactored my code for the rest of the month and opened a new PR.

### July
I switched to the second task on the `mdo(1)` tool. This was a relatively easy task. I had to mainly add **new functionalities** and features to the existing minimal tool. This part felt chill, and after about ten days or so, I opened a PR for this task.

My mentor gave an extensive review of the first task’s PR. Issues: memory leaks, **use-after-free** bugs, and `jail mutex` problems causing crashes. It seems a bit easier now, but just a few months back from today, I did not know what the hell I was doing. I had my mid-term evaluations that month, and thankfully, **I passed**. When I had my first call with my mentor in June, I was confident I would fail the mid-term evals. Anyway, I **worked on the issues** pointed out by my mentor, and July ended.

![Midsem completion email](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2gbtaves87uvlk1ur4zm.png)

### August
By now, **75%** of the work was done on **both** tasks. The second task was almost complete, but the first task **haunted** me with a **mysterious bug**. Days and nights blurred as I tried to find it. I was afraid there was less time left since the third task **hadn’t even been touched**. I found the bug, fixed it, and immediately started working on the new task.

My mentor and I discussed the third task, **and I was blank again**. This time, apparently, the task wasn’t difficult, but I was just not exposed to the files that he pointed out during the call, which I **missed** during my proposal. Anyway, I did that task in about two days, working about **sixteen hours a day**, because significant work remained, and only **ten days** remained until the end. In the final week, I wrapped up everything, added **ATF tests**, and fixed **last-minute issues** just before the deadline.

## Looking Back
All in all, this was a wild experience for me. Coming from a **not-so-low-level programming** background, to completing a project that involved knowledge about **deep kernel-level concurrency**, it was a **ride**. This journey was an extremely fulfilling experience, and I felt a **95% expansion** of my brain after completing it.

And yes, I **passed my final evaluations** and successfully **completed** Google Summer of Code 2025.

![GSoC 2025 completion email](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/oljyhkpsaehnhs1ieaio.png)

What did I learn?

1. Orgs can skip GSoC some years, even with a long track record.
2. I got lucky: I picked a project way beyond my league, and learned things the hard way.
3. Maybe don’t slack off during the Community Bonding period.

I’m genuinely thankful to my mentor, **Olivier Certner**, for being patient with my endless mistakes and questions, and to **The FreeBSD Project** for giving me the opportunity to work on something way beyond my comfort zone. Without their support, I wouldn’t have made it through.
