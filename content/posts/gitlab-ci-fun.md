+++
date = '2026-08-28T23:09:33+02:00'
draft = true
title = 'Gitlab CI and daily routines'
description = "A simple and fun gitlab ci pipeline that everyone should do daily."
tags = ['gitlab', 'ci', 'pipeline', 'fun']
+++

I was reading a senior devops engineer's gitlab ci pipeline and I was amazed by the complexity of it. It was a very well written pipeline with many stages and jobs, but it was also very complex and hard to understand.
I couldn't stop thinking about how to make it simpler and more fun. So I decided to write my own gitlab ci pipeline, with a focus on simplicity and fun.

So, I wrote this weird and simple gitlab ci pipeline that everyone have to do daily. 
We brush our teeth, shower and eat. So tadaa... :D

when my pipeline looks like this, repo: 
https://gitlab.com/shirlenelss/gitlab-ci-project

I install glab-local an npm package that allows me to run gitlab ci pipeline locally.
![gitlab-ci-local](/assets/images/gitlab-ci-fun.png)


That's to see even without pushing any changes to gitlab. 
So it's essential to have while developing. 

![gitlab-pipeline](/assets/images/failing-pipeline.png)

Here's the dependencies that's shown pretty clearly that I have no toothbrush :D 
![gitlab-dependancies](/assets/images/ci-dependencies.png)

And the logs :
![gitlab-console](/assets/images/gitlab-console.png)

I'll continue to play around with gitlab ci. 
Runners, caching, artifacts, and all the other fun stuff.
Look forward to my next post on this subject. 
