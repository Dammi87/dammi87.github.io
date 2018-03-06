---
title:  "Building with Docker!"
date:   2018-03-05 21:04:23
categories: [Docker]
tags: [python, compile]
---
Dockers! What a beautiful invention I have to admit! I really love the fact I can spin up a VM, play around with it, and then completely discard my changes. I especially love this since I've more often than not destroyed my Linux installations with bloatware and experiments.

I started using Docker containers with Tensorflow. Having an easy way of changing between Tensorflow versions on the fly was a great help, and by using `nvidia-docker` as well allowed me to quickly start training using a GPU, all without the hazzle of installing the different CUDA versions needed!

Now, after using Docker for quite some time, I found it often tedious for use and wanted to streamline the way of working with them, especially with teams of people. To explain the thought process, I want to go through some Docker basics first, but I won't actually go into any other details about Dockers. Please clone the Docker-Examples repo if you are interested in following along. 

__Note__: In the following sections I will talk about the different ways one can run a python project within a Docker container. Each section is named after the subfolder from the repository and the scripts should be run from there.

## 1. ADD
A common approach for running an application from within a Docker container is to actually include it inside the Docker image. Consider an application which should print the content of a textfile. The project root looks as follows:

``` console
+-- 1. Add
|   +-- basic.Dockerfile
|   +-- read_this.txt
|   +-- my_reader.py
```
The Docker image implements the `ADD` command, as in it will __copy__ the content of the project into the image at build time.

``` docker
FROM python:3.4-slim
ADD . /app
WORKDIR /app
CMD ["python", "my_reader.py"]
```
We build this container by running `docker build -t examples:1 -f basic.Dockerfile .` and when building is done, we run it with `docker run examples:1`

``` console
usr@usr:~/repos/1.ADD$ docker run examples:1
I'm a text file
This is the second line
There is no fourth line
usr@usr:~/repos/1.ADD$
```
Now, lets edit the text file, and add a fourth line to it and run again:

``` console
usr@usr:~/repos/1.ADD$ docker run examples:1
I'm a text file
This is the second line
There is no fourth line
usr@usr:~/repos/1.ADD$
```
This is because the docker image we built has the old copy of our application stored. We will have rebuild it again to make this work.

``` console
usr@usr:~/repos/1.ADD$ docker build -t examples:1 -f basic.Dockerfile .
Sending build context to Docker daemon  4.096kB
...
Successfully tagged examples:1
usr@usr:~/repos/1.ADD$ docker run examples:1
I'm a text file
This is the second line
There is no fourth line
The third line is a liar
usr@usr:~/repos/1.ADD$ 

```
As you can imagine, this can often become quite tedious, but alas, there is a way around this!

## 2. Volume
The way we can make our changes immedietly noticed by the docker container, is to mount our project root into the running





[Docker-Examples]: https://github.com/Dammi87
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
