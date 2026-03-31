# Understanding Docker Deeply — Not Just Memorizing Commands

If you've ever used Docker, your first time probably went like this. You clone a project, someone tells you to just run `docker compose up`, and either it magically works or it doesn't. And in both cases, you have no idea why.

So you start memorizing commands. `docker build`, `docker run`. You copy Dockerfiles from the internet, and things mostly work until they don't, and then you feel lost because you don't understand what Docker is doing.

This journey walks through the core concepts behind Docker. Not just the commands, but the ideas — so that Docker makes more sense to you.

We'll start with Virtual Machines to understand what problem containers were designed to solve, then dive deep into what containers actually are at the kernel level, and finally map every Docker command back to these fundamentals.
