---
title:  "Building with Docker!"
date:   2018-03-05 21:04:23
categories: [Docker]
tags: [python, compile]
---
Dockers! What a beautiful invention I have to admit! I really love the fact I can spin up a VM, play around with it, and then completely discard my changes. I especially love this since I've more often than not destroyed my Linux installations with bloatware and experiments.

I started using Docker containers with Tensorflow. Having an easy way of changing between Tensorflow versions on the fly was a great help, and by using `nvidia-docker` as well allowed me to quickly start training using a GPU, all without the hazzle of installing the different CUDA versions needed!

## Why I made this
After using Dockers for quite some time, I found it often tedious for daily use and wanted to streamline the way of working with them, especially with teams of people. For example, say that you have a project that will be using basic python for printing a text file to the terminal. The Dockerfile would look something like this

``` docker
FROM python:3.4-slim

ADD . /app
WORKDIR /app
CMD ["python", "my_reader.py"]
```

And my_reader.py simly reads a local textfile

``` python
if __name__ == "__main__":
    with open('read_this.txt', 'r') as f:
        for line in f:
            print(line.replace('\n', ''))
```

To run this, we would build the dockerfile `docker build -t examples:1 -f basic.Dockerfile .` and run it `docker run examples:1`. This is pretty straightforward. Now, everytime we change something in the repository, we would have to re-build the image for the changes to take effect. To get around this, we could simply mount the repository when we spin up the container, `docker run -v  /path/to/repo:/app examples:1`. This will overwrite the old copy of the repository in the image, with the current version, we have one line of code!

Now, assume that the read_this.txt resides in a folder not from your project folder, rather in a folder called /data. Our my_reader.py now becomes

``` python
if __name__ == "__main__":
    with open('/data/read_this.txt', 'r') as f:
        for line in f:
            print(line.replace('\n', ''))
```

We need to grant the container access to this folder as well, at the place where it expects it. Our run command becomes a bit longer, `docker run -v  /path/to/repo:/app -v /data:/data examples:1`.

Assume now, that we have created another script called my_webserver.py which will create serve a webpage, on a specific port. The command becomes `docker run -v  /path/to/repo:/app -p 8080:8080 -v /data:/data examples:1 python3 my_webserver.py`. You can see that the command quickly starts becoming quite large (docker-compose can help here, but it's not applicable for what I wanted to achieve) and especially cumbersome to share the correct command between team-mates, for example:

Bob wants to run this image on his new changes, but he stores his text file at /bobbys_data instead of /data and he doesn't want to run it on port 8080 either. So he has to change the docker command accordingly `docker run -v /path/to/repo:/app -v /bobbys_data:/data -p 6060:8080 examples:1 python3 my_webserver.py`. This in it self is fine, but I wanted a more structured way of doing this, especially when working in a team of people, so I created _build_with_docker_.

## Build-with-docker
Build with docker is a wrapper around your project, which will read user-specific build-commands, mount folders, open ports etc with a very simple command. The command would look the same for me and Bob, `bwd my_webserver.py`. And whats even better, you can setup a remote computer to which you have SSH access to and have your project built there instead, with almost the exact same command.

### Prerequisites
Detailed installation instructions are found on the project [Github page][bwd]

Your project folder structure should have the following structure
``` txt
+-- Project_root
|   +-- Dockerfiles
|	+--+-- some_name.Dockerfile
|   +-- bwd.json
|   +-- ...
|   +-- ...
```

Mainly, it should have a folder that contains your project related Dockerfiles and a bwd.json config file. In the example we talked about, the config file would look like so

``` json
{
    "adamf": [
        {
            "build_name": "regular_build",
            "docker_file" : "some_name",
            "volumes": ["/data"],
            "ports": ["8080:8080"],
            "build_cmd": "python3 -u",
            "run_as_module": false
        },
        {
            "build_name": "another_build",
            "docker_file" : "some_name",
            "volumes": ["/data"],
            "ports": ["7231:8080"],
            "build_cmd": "python3 -m",
            "run_as_module": true
        },
    ],
    "bob": [
        {
            "build_name": "bobs_build",
            "docker_file" : "some_name",
            "volumes": [["/bobbys_data", "/data"]],
            "ports": ["6060:8080"],
            "build_cmd": "python3 -u",
            "run_as_module": false
        }
    ]
    "common" [
        {
            "build_name": "build_on_remote",
            "docker_file" : "some_name",
            "volumes": [["/remote_data_folder", "/data"]],
            "ports": ["8000:8000"],
            "ssh_ip": "localhost",
            "ssh_user": "YOUR_USER_NAME", 
            "remote_folder": "/remote_build_folders/users/",
            "build_cmd": "python3 -u",
            "run_as_module": false
        }
}
```

### Running locally
You simply need to run the `bwd my_webserver.py` from the project root, and it will automatically create the necessary command for us. It looks at the username that is running the command, look for it in the build config and run. It is possible to define multiple build commands, and choosing between them by explicitly specifying which build to use `bwd my_webserver.py -build_name another_build`, if none is specified, it defaults to the first one.

But here comes the fun part, we can now run our progam easily on a remote host. If you followed the instruction regarding SSH on the [GitHub][bwd] page you can try this as well. In this example our "remote" is simply our own computer for demo purposes.

### Running on a remote
When working with a team in a deep learning project, sharing GPUs on the same machine can often be quite difficult. Using nvidia-docker it's quite easy to "hide" other GPUs from the container, allowing a clean seperations between them. At one point we even had 4 different training sessions on a computer containing 4 GPUs without much difficulty. 

In this simple case, there is no need for GPUs, but lets say we are interested in making our script hosted on a remote-computer of ours. Running the command `bwd my_webserver.py -build_name build_on_remote` will copy all project files, send them through an ssh tunnel at YOUR_USER_NAME@localhost to `/remote_build_folders/users/$YOUR_USERNAME/PROJECT_NAME` and run the docker command on the remote host. And whats even better, is that your teammates can do the same thing with the same command (all users have access to the "common" part of the build config).

Another thing that compliments running dockers on a remote-machine is if you setup a [Portainer][portainer] on it, you can monitor and kill any running containers from a website. 

## Sublime
I now use this everyday with Sublime. I'm currently working on a Sublime plugin for this, so that the build-systems defined in the config file become visible in the build menu of sublime, but fright now I have a simple build command the runs the first build-setting for my user name.

If you are using Sublime, simply go to Tools - Build System - New Build System and paste in the following: 
``` json
{
    "cmd": ["bwd", "-proj", "$project_path", "-s", "$file"],
    "file_regex": "^[ ]*File \"(...*?)\", line ([0-9]*)",
    "selector": "source.python"
}
```

I'll update this part of the blog-post when I've finished the plugin.

[bwd]: https://github.com/Dammi87/build-with-docker
[portainer]: https://github.com/portainer/portainer