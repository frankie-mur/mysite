+++
title = 'Microservices are Techinal Debt'
date = 2024-09-28T12:37:58-07:00
draft = false
tags = ['microservices', 'system design']
+++

# Microservices

I recently watched a video about microservices titled [Microservies are Techincal Debt](https://www.youtube.com/watch?v=LcJKxPXYudE&t=2s)
it piqued my interest in the topic. As a junior engineer, you are taught that microservices are a good thing.

- Monoliths = Bad

This video made me take a step back and ask why.

## Why Microservices

Microservices solve many problems, some major ones are

1. Domain focused
2. Faster deployments
3. Easier to handle scale (scale up or down pods)
4. Less friction/blockers between teams (this is a big one I have come to find out)
5. Loosely coupled
6. Data Ownership
7. Programming language agnostic, individual teams can pick a language

## Microservies are Tech Debt?

The interview makes the point that microservices are technical debt, Here is why

- While the initial adoption of microservices allows teams to move faster, eventually that will come to a halt.
- Most microservices will depend on their dependent services, creating a Micro-Monolith.
- Microservices is a socio-technical problem, it safely allows engineers to code on their domains

## Take Away

The age-old safe answer here I believe is, "It Depends". No solution is the silver bullet right answer, there are always trade-offs
for any solution and I think that is the way it should be. I agree that microservices in a way always end up being somewhat of a
"Micro"-"Monolith", in my company we always deal with dependent services failing, which ends up causing issues upstream. In a way,
all services depend on each other.
