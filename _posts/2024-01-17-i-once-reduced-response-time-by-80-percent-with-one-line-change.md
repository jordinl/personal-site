---
layout: post
title:  I once reduced response time by 80% with one line change
date:   2024-01-17
image: /assets/img/memory-fix.png
---

Shortly after I started a job I noticed we were storing, for each of our customers, Stripe customer data as JSON in 
the DB. I actually said in a meeting that I thought it would be better to just store only the attributes we needed 
as columns instead, but I was told _not to worry about it and that it worked beautifully_. It was 
actually worse than that because Stripe allows to expand responses when querying their API so we were storing 
customer data with their subscriptions and payment method metadata; after that when we wanted to read this 
information we would hydrate Stripe 
objects from the [Stripe ruby gem](https://github.com/stripe/stripe-ruby).

So a few months go by and I'm asked to investigate some memory issues on production. I start chasing down the issue 
on my development machine, run some memory profiling and notice the Stripe gem is doing a lot of memory allocations. 
Thought that couldn't be the issue so I start looking somewhere else. The actual pages with the biggest memory 
usage allowed embedding custom images and CSS so I thought the issue would be there. I spent 
some time looking into that, but I couldn't find what was the cause. So I started commenting out one piece of code at 
a time until I noticed it was those Stripe JSON objects that were causing these huge memory spikes. 

Everytime we wanted to check what type of Stripe subscription the customer had or if it was active we would hydrate 
a Stripe object with the subscription data from the DB. The temporary fix I did was to memoize the hydrated Stripe 
subscription so this inefficient process was done only once per request and not many times, I think I counted 6 
times in a single request.

On my development machine I noticed an 84% reduction in memory allocation for certain requests, so I knew it would 
improve 
things. When 
we pushed the code to production I couldn't believe what I was seeing. I was looking at our APM tool to check 
performance and had to refresh the page a few times because I thought it wasn't working right. Mean response time 
had dropped 80%, it also looked flat now instead of having lots of peaks and valleys. 

<p class='flex'>
  <img src='/assets/img/memory-fix.png' alt='Memory Fix' />
  <small>
    The pink solid line represents new response time and the pink dotted line old response time
  </small>
</p>
