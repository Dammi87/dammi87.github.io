---
title:  "Building with Docker!"
date:   2018-03-05 21:04:23
categories: [Docker]
tags: [python, compile]
---
Docker! What a beautiful invention I have to admit! I really love the fact I can spin up a VM, play around with it, and then completely discard my changes. I especially love this since I've more often than not destroyed my Linux installations with bloatware and experiments.

I started using Docker containers with Tensorflow. Having an easy way of changing between Tensorflow versions on the fly was a great help, and by using `nvidia-docker` as well allowed me to quickly start training using a GPU, all without the hazzle of installing the different CUDA versions needed!

## Why I made this
After using Dockers for quite some time, I found it often tedious for daily use and wanted to streamline the way of working with them, especially when working in a team. For example, say that you have a project that will be using basic python for printing a text file to the terminal. The Dockerfile would look something like this

``` docker
FROM python:3.4-slim

ADD . /app
WORKDIR /app
CMD ["python", "my_reader.py"]
```

And my_reader.py simply reads a local textfile

``` python
if __name__ == "__main__":
    with open('read_this.txt', 'r') as f:
        for line in f:
            print(line.replace('\n', ''))
```

To run this, you would build the dockerfile `docker build -t examples:1 -f basic.Dockerfile .` and run it `docker run examples:1`. However, everytime you change something in the repository, ypu would have to re-build the image for the changes to take effect. 
To get around this, you could mount the repository when spinning up the container, `docker run -v  /path/to/repo:/app examples:1`. This will overwrite the old copy of the repository in the image, with the current version.

Now, assume that read_this.txt resides in a folder not from your project folder, rather in a folder called /data. Your my_reader.py now becomes

``` python
if __name__ == "__main__":
    with open('/data/read_this.txt', 'r') as f:
        for line in f:
            print(line.replace('\n', ''))
```

However, the file resides within a folder that the docker container is not aware of, you need to mount that folder into the docker container as well. Thus, your run command becomes a bit longer, `docker run -v  /path/to/repo:/app -v /data:/data examples:1`. 

Assume that you have created another script called my_webserver.py which will serve a webpage on a specific port. The command becomes `docker run -v  /path/to/repo:/app -p 8080:8080 -v /data:/data examples:1 python3 my_webserver.py`. You can see that the command quickly starts becoming quite large ([docker-compose][docker-compose] can help here, but it's not applicable for what I wanted to achieve) and especially cumbersome to share the correct command between teammates, for example:

Bob wants to run this image on his new changes, but he stores his text file at /bobbys_data instead of /data like you do, and he doesn't want to run it on port 8080 either. So he has to change the docker command accordingly `docker run -v /path/to/repo:/app -v /bobbys_data:/data -p 6060:8080 examples:1 python3 my_webserver.py`. This in itself is fine, but I wanted a more structured way of doing this, especially when working in a team, so I created _build_with_docker_.

## Build-with-docker 
Build with docker __(bwd)__ is a wrapper around your project, which will read user-specific build-commands, mount folders, open ports etc with a very simple command. The command would look the same for me and Bob, `bwd -s my_webserver.py`. And what's even better, you can setup a remote computer to which you have SSH access to and have your project built there instead, with almost the exact same command.

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

Mainly, it should have a folder that contains project related Dockerfiles and a bwd.json config file. In the above example, the config file would look like

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
Run `bwd -s my_webserver.py` from the project root, and it will automatically create the necessary command. It looks at the username that is running the command, searches for it in the build config and runs. It is possible to define multiple build commands, and choosing between them by explicitly specifying which build to use `bwd -s my_webserver.py -build_name another_build`, if none is specified, it defaults to the first one.

As an added bonus, it is possible to run the progam on a remote host. If you followed the instruction regarding SSH on the [GitHub][bwd] page you can try this as well. In this example the "remote-host" is simply your own computer for demo purposes.

### Running on a remote
When working with a team in a deep learning project, sharing GPUs on the same machine can often be quite difficult. Using nvidia-docker it's quite easy to "hide" other GPUs from the container, making them seperate from one-another. At one point I even had 4 different training sessions on a computer containing 4 GPUs without much difficulty, all in a seperate container.

In this example above, there is no need for GPUs, but lets say you are interested in running the script on a remote-computer of yours. Running the command `bwd -s my_webserver.py -build_name build_on_remote` will copy all project files, send them through an ssh tunnel at YOUR_USER_NAME@localhost to `/remote_build_folders/users/$YOUR_USERNAME/PROJECT_NAME` and run the docker command on the remote host. And what's even better is that your teammates can do the same thing with the same command (all users have access to the "common" part of the build config).

Another thing that compliments running dockers on a remote-machine is if you setup [Portainer][portainer] on it, you can monitor and kill any running containers from a website hosted on the remote itself.

## Sublime
To be honest, I'm a fan of Sublime and use it everyday with _bwd_. I'm currently working on a Sublime plugin that is capable of reading the bwd.json config, so that the build-systems become visible in the build menu of Sublime, but for now I have a build command that is equivalent of running `bwd -s /relative_path/to/script_to_be_run.py`

If you are using Sublime, go to Tools - Build System - New Build System and paste in the following: 
``` json
{
    "cmd": ["bwd", "-proj", "$project_path", "-s", "$file"],
    "file_regex": "^[ ]*File \"(...*?)\", line ([0-9]*)",
    "selector": "source.python"
}
```

I'll update this part of the blog-post when the plugin is complete.

[bwd]: https://github.com/Dammi87/build-with-docker
[portainer]: https://github.com/portainer/portainer
[docker-compose]: https://docs.docker.com/compose/