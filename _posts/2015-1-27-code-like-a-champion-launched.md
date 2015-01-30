---
layout: post
title: Code Like A Champion Launched
tags: general
description:  The birth of Code Like A Champion including the goals of this blog and how it got its name.
---
Almost a year since I first installed Jekyll, followed by design work by a now former colleague of mine, and recently many hours of hacking on stylesheets, I'm happy to finally officially launch *Code Like A Champion*, a blog focused on fundamentals of system design and software development practices particularly in the .NET arena.  There's thousands of such blogs on the Internet (maybe millions): what makes this one special?  Knowledge, skill, and talent can carry you a long way, but there's one attribute more important than any of those: coding like a champion. 

### What does it mean to be a champion? ###

So I'm sure this question has struck you by now based on the blog name:  what does he mean be a champion and isn't that the same macho attitude he just railed against?  Being a champion has nothing to do with arrogance or intolerance, it isn't a function of rich or poor, of a high title or an entry-level one, of being an expert in a field or a novice.  And most importantly, being a champion is not a matter of wins and losses.  

Well then, what is it?  Let's hear from some experts in the field first.

<blockquote>"We will either find a way, or make one." - Hannibal, <em>Carthaginian general</em></blockquote>

<blockquote>"The spirit, the will to win, and the will to excel are the things that endure. These qualities are so much more important than the events that occur."  - Vince Lombardi, <em>legendary football coach</em></blockquote>

<blockquote>"The greatest reward for doing is the opportunity to do more." - Jonas Salk, <em>discovered first polio vaccine</em></blockquote>

<blockquote>"The successful warrior is the average man, with laser-like focus"
 - Bruce Lee, <em>champion martial artist and actor</em></blockquote>

<blockquote>"Start where you are.  Do what you can.  Use what you have."
- Arthur Ashe, <em>tennis player</em></blockquote> 

<blockquote>"We are what we repeatedly do. Excellence, then, is not an act, but a habit."
-Aristotle, <em>philosopher</em></blockquote>

<blockquote>"Accept the challenges, so that you may feel the exhilaration of victory." - <em>General George S. Patton</em></blockquote>

<img src="{{ site.url }}/images/winners-vs-losers.png"  />


### I don't play sports for a living, how does this help me? ###
 
Well first, most of the quotes above were not from a sports figure at all.  Hannibal, the famous general, led his men from Carthage (in north Africa) through Europe and into Italy where he spent years fighting Rome.  Rome, the greatest city on Earth, sought Hannibal's destruction, but even with him thousands of miles from home, they were unable to defeat the great general and his army (at least directly).  Hannibal led his men on a march through the freezing Alps en route to Rome because he refused to be conquered and because he instilled this spirit and trust in his men.  Jonas Salk discovered the first polio vaccine.  For most people, that's a lifetime achievement, an opportunity to rest on one's laurels.  But not to Salk, who pressed forward to eradicate polio and turn his attention to other conditions.  

### Okay, okay, I get it.  So what's a champion software developer? ###

Being a champion in any field, to me, is about one thing: attitude.  I have known a lot of incredibly talented people who were thwarted at every turn by their own attitudes.  Attitude in this context includes a lot of things.  Some people pride themselves on being "realists" and "telling it like it is", at least in their own minds.  But it generally takes a minimal amount of skill to point out the problems and issues with something.  And it takes less skill than that to continually harp on things and complain.  But champions view problems as opportunities:  an opportunity to do something new or different, an opportunity to push the boundaries.  

Sometimes attitude is in the view of one's job, often expressed as "I do X and you do Y."  But in the software development teams I've been on, ultra-specialization is a negative, not a positive.  Yes, there is definite value in knowing something at a deep level, but I'm talking about someone who will only do one particular job with no mind to go outside of it.  In the Agile teams that are more and more common today, there is little room for someone not willing to go outside their current skillset, but I see it too often.  Why are otherwise smart people so resistant to this?  Lot of reasons, but the most common one I've observed is the fear of being a novice.  Expertise in something is hard won and many people are loathe to give it up, to return to being a beginner, to struggle and to risk a perception that they're not catching on fast enough or that maybe their initial success was all a sham. But the world today is different from the world yesterday; today demands new skills and if we don't develop them, someone else will come along that has those skills and we will be out of a job.  Instead, a champion looks for opportunities to grow their skills and learn something new, to risk temporarily looking foolish to come out with more skills.  A champion must improve every day:  every day we must be better than the last.  

Finally, a common attitude I see is that one's work environment is inflicted or imposed upon you by "management" instead of something to be shaped, expressed as "Well management will never let us write more unit tests" or "I understand what you're saying about code reviews, but management won't let us take the time."  But if pressed, usually "management" has not communicated any such thing, but rather people are either afraid to push for change or afraid to take a stand.  Yes, sometimes management will decide things that are contrary to what we, on the ground, might want or think best.  But that does not remove the obligation to identify practices and approaches that would represent meaningful value and to push for their adoption.  You miss 100% of the shots you don't take and an idea you don't propose will not be adopted.  It takes courage to speak up, to introduce an idea, to offer it up for criticism by peers.  But a champion transcends that fear.    

### I'm ready to be a champion.  What will that mean for this blog? ###

In years I've spent training in-house developers, acquiring skill in a new language or pattern is about one thing: changing one's mindset.  The best example I can offer is SQL.  There is a lot of really awful SQL out there.  And why is that?  SQL isn't a particularly difficult language to learn syntactically.  It's pretty easy to put together simple queries and get some results.  But why is so much of it awful?  Well typically, developers learn SQL after they already know one or more imperative or object-oriented languages.  But SQL is neither of those:  it is declarative.  The SQL query engine exists to parse your query and convert it into physical operations.  Too many developers write SQL as if they're (consciously or subconsciously) trying to command the engine to compose the results the way they would manually.  Learning to write efficient, beautiful SQL is about one thing:  changing one's mindset to the language of sets.  Instead of instructing the query engine how to gather the results, SQL should be written to simply describe what should be true about elements in the result set; let the query engine figure out how to put it together (that's why it's paid the big bucks, after all).  

Learning to be a champion also requires a shift in mindset:
* We must be mindful of our day to day practices and improve them continuously
* We must grow our skills as if our lives depended on it
* We must solve problems, not just point them out
* We must take a stand for things we believe in
* We must raise up our teammates
* We must make commitments, not promises

Since coding like a champion does require knowledge and skill, this blog will also focus on fundamentals in system design and software development that often go uncovered or with too little depth and explore them more fully with code examples and suggested resources. There are a lot of great resources out there (blogs, books, videos, training courses), most or all written by people far more talented and knowledgeable than me and I'm no replacement for those at all. For newer .NET developers, there is a gap between there current skills and knowledge and what's needed to grok a lot of those materials and for a lot of developers across skill levels, attitude and mindset are the limiter. I merely hope to share some ideas and approaches that can help anyone, starting from where they are right now, to code like a champion.   


