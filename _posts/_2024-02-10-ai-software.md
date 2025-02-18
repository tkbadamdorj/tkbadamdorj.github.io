---
layout: post
title: AI as just another program
date: 2025-01-20 15:09:00
description: How to give in to your fears of SQL and/or possibly improve your life
tags:
  - opinion
categories: 
featured: false
---
When we're writing code, we don't worry about bits. We don't worry about how data is laid out in memory, or which registers to use.

We just write code knowing that those details are taken care of. What we do care about is whether what we wrote makes semantic sense.

From an engineering perspective, how is AI any different? 

It's a learned program but it takes as input something, outputs something, and we can test it for correctness. 

Maybe in the future, instead of training algorithms how we do them now, we write something like:

model = classify_cats_and_dogs()
assert accuracy(model) > 90

and we have soft asserts like this 
where we don't even think about the learning process - or maybe models are truly plug and play 

we have soft code like this that does things for us

