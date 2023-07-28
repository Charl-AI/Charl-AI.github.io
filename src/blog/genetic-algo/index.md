---
title: Automating my Coursework with a Genetic Algorithm
subtitle: AI does my machine learning coursework for me! Achieve the singularity with this one weird trick! 
date: 2021-08-28
word_count: 1035 words ~5 minute read
---


I took a module on machine learning in my final year of my MEng degree. It was an enjoyable module on the basics of machine learning but it happened to have a rather boring piece of coursework. This coursework simply involved designing and training a simple feedforward neural network to solve a classification task (predicting performance of catapaults from design parameters). This wouldn't be so bad, however, the work was marked based on both the performance of the model on an unseen dataset **and** the *simplicity* of the network (fewer parameters is better) - essentially turning the coursework into an architecture search task. Most of my friends simply manually searched architectures until they found one they liked, however, this is boring, repetitive and may not even yield a great answer. This project was my attempt to automate this boring coursework and solve the problem elegantly.

The project source code can be found [here][repo-link].

## Principle

Searching neural network architectures manually is boring and time-consuming, so I wished to automate the task. Grid search and random search methods are often popular methods for tasks like this, however, they can be inefficient (especially when the search is in a high-dimensional space). Evolutionary algorithms are an intuitive and fun way to solve this problem, inspired by natural selection. First a population is initialised and individuals are ranked based on fitness and passed through a selection algorithm. Next, the selected individuals breed (by crossing over genes) to create a new generation (with some random mutation in the genes). This process is repeated for a set number of generations, (hopefully) resulting in a final population of strong individuals.

The implementation details of my method are described in the [repo][repo-link], including describing the 'genes' given to the models, how I measured fitness, my crossover and mutation algorithms, as well as other concepts I implemented such as speciation and elitism.

## Results and Summary

In total, 1000 models were trained over 10 generations. The gif plots the location of each model in the hyperparameter space, with the color representing the loss of the trained model (lower is better) and different marker shapes being used for each species. The gif shows the evolution of the different species over the generations and makes it clear how different species find different strategies and form distinct clusters (and is quite mesmerising to watch!). It's also interesting to note how one species managed to wander away from the hyperparameter space that I had predefined by utilising the Gaussian mutation mechanic. This could be considered a bug, but I like to think of it as proof that the algorithm is able to find hyperparameter combinations that I would not have considered to be good!

The final architecture achieved an accuracy of 88.2 % on the seen dataset and 85.9 % on the unseen dataset, so overfitting was low. The overall model had 713 parameters which is a very concise model for this task. Overall, the model demonstrated a strong performance and was awarded an A+ grade.

![](src/blog/genetic-algo/assets/evolution.gif)

Despite the seemingly strong performance of the algorithm, I ended up concluding that this is actually not a particularly efficient method of neural architecture search, requiring around 1000 models to be trained to find only a modest improvement over hand-designed architectures. This problem would be exacerbated further in computer vision tasks, where models train on GPU and typically take much longer.

Nonetheless, this project was a great experience for me - I replaced a boring and repetitive task with some coding and ended up learning even more about machine learning! Looking back on this project though, I think the real lesson to be learned is actually nothing to do with AI and more about software engineering and philosophy...

## The real lesson here: code quality and design patterns

There's no getting around it... the code I wrote for this project is horrible. I almost want to delete it from my GitHub to pretend it never happened. The project was whipped together pretty much in an afternoon and, while making it, I was hoping to not end up spending longer trying to automate my coursework than I would have done actually doing the coursework. Still, this code is bad. What was I thinking?

Well, partly it is because I was a worse programmer then. I have come a long way in the year or so since I made this project and I now know things that I didn't back then. I will (I hope) improve even more in the coming year and, in a years time, will be thinking the same about my current projects. But there's another aspect to this. It's not just that I didn't know how to write perfect code then, it's that I *chose* to write bad code. In my warped thinking at the time, I conflated quickly writing bad code with being able to finish the project quickly. *I'll just whip this up as a proof of concept so I wont worry about writing nice code* was my thought process. This is, obviously, wrong. Writing good code helps you debug, add features, and change your ideas quickly and painlessly - in the end, it always saves you time in the long run. Everyone knows this! So why didn't I do it at the time? Honestly, I don't know, all I know now is that I've got a decent project with a codebase I'm ashamed of - I don't want to touch it again and I can't recommend you do either. **Writing bad code leaves a bad taste in everyone's mouths.**

So, join me solemnly swearing never to deliberately write bad code again. I will make mistakes, but they will be through ignorance, not choice. Lead me from temptation and deliver me to good practice. *Amen*.

[repo-link]: https://github.com/Charl-AI/Genetic-Hyperparameter-Optimisation
