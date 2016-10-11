+++

title = "The Perfect Startup Culture"
date = "2016-10-11T13:16:30+05:30"

categories = ["startups", "culture"]

socialsharing = true
totop = true

author = "Goutham Veeramachaneni"
authortwitter = "https://twitter.com/putadent"

notoc = true
+++

# Prologue

It is internship period here at IIT Hyderabad and I was sitting in the PPT (Pre-Placement Talk) of a fast moving product based Startup: [Arcesium](http://www.arcesium.com).

For the most of their presentation they focused on the right thing: **Company Culture**. Their culture was almost exactly similar to the company I interned in during Summer 2016: [Boomerang Commerce](http://www.boomerangcommerce.com). That internship was the best time of life and I regularly miss being there.

That's when I realised, while there might not be a perfect startup culture, there are certain qualities which every good startup has (and are extremely important for speed and success?). While some may feel that some of these are not needed, these will be the most important qualities in any startup I start or join.

This article is primarily for me to remember to make the right choice but I hope this will help others too in building their startups/careers.

----

# The Qualities

I am listing down qualities in no particular order. I will also try to show real examples I experienced which correlate with the qualities and the impact they had.

## Performance Driven

A startup should purely be performance driven. If someone is doing well, promote him. The promotion need not be an actual move up the ranks. It could be more autonomy, more ownership. Just give him extra space and freedom if he is good, and stand back to watch magic happen. People just love it when they realise their efforts are paying off and will start to work more!

I got right to code at Boomerang and I tried to eat up everything they threw at me. While initially the zeal of just joining made me work hard, I slowly saw that I was getting more autonomy and that made me feel good inside.

Some time later when I proposed that I implement a monitoring solution (Something BIG and useful), I was immediately given the green signal and COMPLETE AUTONOMY! This made me feel responsible and trust me, I worked on it as if my life depended on it. When it finally worked stably, the feeling of accomplishment and pride I felt are inexplicable. It was *my* system and a lot of people were now using it!

It was clear that I got all that freedom because of the work I had already shipped. This freedom and granting it are both  equally important.

## Amazing problems to wake up to

I was lucky to have woken up excited to get to work almost everyday. I check the status on my laptop and then proceed to brush my teeth. That's because I was not going to do trivial work which I already knew. I was fixing things no one else faced. I was learning tons of new things everyday.

Take the case of the Prometheus deployment. Nobody had seen such scale and churn in services. We were running 30K services which have very high churn (say, all of them are recycled every 2 hours on average). And the datamodel of Prometheus does not like this. It was crashing frequently and I was trying a lot of fixes.

I was learning a lot about filesystems and even reformatted the filesystem on AWS to have more inodes (We ran out of inodes due to large number of files created by Prometheus due to the churn). This is just one fix I tried and it literally took me 2 weeks to try tons of other stuff and I could finally fix it by writing a high performance service in Golang. Now this scenario had me waking up early to check on Prometheus' health, replies on IRC, github tickets, etc.

Now, this was just one case but I was fixing and learning so much that I wanted to spend all my time doing just that! Knowing that what you're doing is great will motivate you to great lengths.

## Appreciate the good work

We regularly get mails like this that make you feel good. The most recent being:

![Good Work](https://i.imgur.com/RzveVAM.png)

This is just one example and there are a lot of other ways (implicit and explicit) in which good work is appreciated in Boomerang. And if you are at the receiving end of it, trust me, it feels amazing. Not only the one who receives it but the others around them do too.

For example, we have a quarterly performance thingy where one person gets an awesome gift. While I was there, it was a Nexus 6 and it went to Gaurav from my team and, trust me, the whole team felt good!

Finally, I am pretty sure that small hints of appreciation pushed me to write more polished solutions sometimes :)

## Infinite learning / exploration opportunities

At Boomerang we were fixing hard problems and any good work is appreciated. But sometimes, you do stuff because you want to try something new, or to pick up a new skill. Any good startup should accommodate that if they expect their employees to perform well.

Take the case where we were mapping products to their variants on a large scale. It would take 22-24 hrs, but because it was part of the normal flow and is a routine job, nobody noticed/complained. Gaurav wanted to optimise that and wanted to use AWS Elastic Map-Reduce (EMR) to do that. He had never used EMR before and was very keen on doing it. I am pretty sure all it took for him to gain access was an email. Once he got access, he immediately got to work and reduced the job time to around 2 hours. I am confident that a large part of his motivation was to pick up EMR!

Even in my case, when I wanted to try AWS ML, all it took was an email and 15 minutes to get access. Nobody else had access, but again, that was not an issue. I primarily wanted to check out how good it was. :P

## Appreciate and accommodate mistakes too

This could very well be my most important takeaway from the internship. While you will find most of the other qualities I've mentioned in other articles too, this one is rare. Startups should understand that mistakes happen and should create an environment where mistakes of any degree could be acknowledged, fixed and, possibly, even appreciated.

Take this particular situation where we had a choke check script (to check if every service was running fine) with a small typo causing us to restart our worker services every 5 minutes! While this made things inefficient, it did not bring the infrastructure down as we had fail-safes in place to prevent workers from crashing. I was looking into something else entirely with Mohammad (An amazing dev, and the person who authored the script) when we realised that this bug had in fact crept in a couple of months prior to this. We promptly fixed it, now what he did next surprised me: He called out the mistake to the whole team (We sit together) and the teams reaction was "Wow, really?! Damn, at least we fixed it now."

This incident is seared into my memory. I don't remember what we were looking into, but I remember him calling out the mistake and the "OK, good" reaction of the team. Probably because at that point in time, I would have fixed it and moved on hoping nobody notices. That small incident made a huuuge impact on me.

This was clear even when we screwed up big time. It was around the end of the intern and I had to wrap up a lot of things. I wasn't feeling particularly well, but I still went to my office and ran a job with some buggy code. It went on to remove some very crucial info from the database and nobody knew why. I had fixed the bug and thought I fixed everything I touched. But this was noticed several days later and nobody had a clue as to why that happened (My script was working fine now and the team wasn't doubting it). When I was thinking about it at home (My intern had ended and I was back), I realised that the initial bug could have been the root cause of all this and I immediately sent an email to my manager Anshul (Another amazing guy). I was afraid of what he would say, but all he said was, "Cool, let's get on a call with the others and lets fix it". This was at 11:00PM and the team got together and fixed it. I haven't screwed up since (I am back at college :P) but I know that I would admit it as soon as I realise it.

When you move fast, you break things. Make sure you realise that. Just create a culture where people can accept and fix mistakes without fear of judgement.

## Growth

This is not something I realised personally, after all I was an intern, so I am not going to spend a lot of time on this. But over several conversations, I realised people actually looked forward to growth in their careers a lot! It was implicit that people enjoyed and stuck on because they were growing very rapidly in Boomerang. They were looking at others being promoted and were personally inspired to achieve the same.

Just make sure the good people grow. Else they won't stick!

--------------

Now come some qualities that are on the people side of the culture and are equally important.

## Team: Will they be your best friends?

Everybody talks about team and, trust me, the team I worked with at Boomerang trumps every other reason here. I just love goofing around with them, playing TT with them and discussing problems and solutions. They are the reason I spent 12 hours in office on average per day.

Boomerang was an extension to college with young people (my team had a lot of 2014, 2015 graduates) and I fit right in. Even though I was an intern, they treated me as an equal and I respected and loved them a lot. I spent a lot (all) of time with them. We used to come to office on weekends too just to hang out!

Being comfortable and enjoying your team's company is critical. It will be a prime factor for coming to office excited. No hesitation to approach/criticize your co-workers increases speed and satisfaction and I cannot stress on this enough.

I miss my team a lot and speaking to them brings back a lot of memories and they're the primary reason I want to go back. I just hope they miss me at least 10% of how much I miss them.

## Flat Structure

I walked into Boomerang and Arafath helped me meet people. When I met Madhu, who runs the India office, I said "Sir". The first thing he said was, call me "Madhu", nobody calls people here "Sir" anyways. I started calling everyone by their first names and I realised it made approaching them with everything (ideas, mistakes, etc.) easier.

Even though there are very few levels, I guess they aren't even visible. The managers sit with the team and if someone walks in, he wouldn't able to figure out who the manager is. Everyone at every level is treated the same! In fact, the only structures visible are the different teams. I bet people would consider Madhu just another employee if they see him in office. And this makes us very comfortable at office.

## Recreational Areas

When you are in a fast moving startup, trust me, you need to take regular breaks. Call it stress busting or taking your mind off a hard problem, you need breaks. And the startup should accommodate for that.

We had 3 primary avenues at Boomerang: a TT table (my turf), a Foosball table and a balcony to smoke. I only played TT (picked it up during the intern) and I played >2hrs of TT on most days! It helped write and debug better.

## Informal environment

The environment at Boomerang is completely informal. Madhu and Anshul wear shorts a lot! We could swear and goof around a lot. Few days into the internship, I went back to my college dressing: crumpled shirts and shorts. Now that I think about it, I think people noticed, but they didn't care!

I think this directly inherits from the performance driven culture, but the ability to be yourself and not care about people judging you is key to employee satisfaction. Now, that is not possible if you have rigid policies about everything under the sun in place.

-----------

## One more thing: Food
This is not essential, but this is very important for me :)

I think Boomerang is the only place away from home where I put on weight, that too given the fact that I was playing > 2 hours a day! Boomerang has a fully stocked up cafeteria which I visited every hour at least.

The noodles, the biscuits, the cake, the Tropicana, the lassi, yumm! I guess I miss the food more than the team now. :P I believe eating will help you think better.

------

I'm done here. So, here is an image of my awesome team! Some are missing though.

<div>
  <img style="max-width: 100%" src="https://i.imgur.com/6Brimfw.gif"></img>
</div>
