---
published: true
layout: post
title: "Octopress in a Docker container"
date: 2017-04-12 22:46:14 +0000
comments: true
categories: Blogging, Octopress, Docker
---

I use [Octopress][octopress] for my, albeit infrequent, blogging. I like the idea of statically generating the blog site with the posts and keeping it all in Git. I also enjoy the ability to quickly type up a post using [markdown][markdown] and then publish it. No need to log in to a site to write the post.

My biggest problem with Octopress is that it requires me to install and manage a [Ruby][rubylang] environment. Either by outright installing Ruby on my machine or through [rbenv][rbenv]. This usually means, for me at least, a struggle to get all the gems successfully installed. Adding plugins often result in errors while installing gems either due to dependencies between gem versions, or some system level dependency that is not available.

These system level dependencies really bugs me so when I recently replaced my laptop I decided to keep my OS Ruby free by managing the Octopress Ruby environment and the system level dependencies in a [Docker][docker] container rather than using `rbenv`


> TLDR; <small>The impatient can find the Dockerfile at: https://github.com/dirkvanrensburg/octopress2-dockerfile</small>

### Tested environment

Thing     | Version
----------|-------
OS        | Ubuntu 16.04
Docker    | 17.03.1-ce
Octopress | 2.0

<br/>

### Assumptions

[Github Pages](github-pages) to publish and host my blog and this post assumes that you are too. However, this post should be useful even if you don't want to use Github Pages.

### Octopress

Now that you have a working container, it is time to do the Octopress installation. You can either start with a fresh copy of Octopress, see the [documentation][octo-docs] or with an existing Octopress blog. If you have an exising blog then you may have to tweak the Dockerfile a bit to add any system level dependencies your blog my have.

#### New Octopress blog

For a new blog you have a bit of a chicken and egg problem. You need to get the Octopress stuff, but you can't run any Ruby dependent commands because Ruby is in the Docker container.

Clone Octopress and take note of the `Gemfile` in the root of the repository: 
```
git clone git://github.com/imathis/octopress.git octopress
cd octopress
```

> <small> You'll need the `Gemfile` in the root of the blog repository for building the Docker image later on. </small>


#### Existing octopress blog

The following steps should get the blog repository in the correct state for generating and publishing.

* If you don't have a current copy of your blog then clone the repository from Github
* Once you have cloned the repository change into the repository directory and change the working branch from `master` to `source`
* Create the `_deploy` folder and change into that directory
* Now link the `_deploy` directory with the master branch of your blog repository
  * `git init`
  * `git remote add origin <githubrepourl>`
  * `git pull origin master`

> <small> You'll need the `Gemfile` in the root of the blog repository for building the Docker image later on. </small>

### Docker

Docker gives you the ability to create a lightweight container for processes and their dependencies. You create a container from some published base image and then install the necessary packages as you would on a normal machine. You can then run the container, which will start the required process and run it without poluting the host operating system with unecessary dependencies.

#### Dockerfile

A `Dockerfile` is a file which tells the Docker daemon what you want in your container. How to write a `Dockerfile` and what you can do with it is extensively documented [here][dockerfile-docs].

You basically start with a `FROM` statement telling Docker where you want to start and then telling it
which package to install. In this case you have a number of dependencies as seen here:

```
FROM ubuntu:16.04

RUN apt-get update -y && apt-get -y install \
  sudo \
  gcc make \
  git \
  vim less \
  curl \
  ruby ruby-dev \
  python2.7 python-pip python-dev
```

These packages should be enough to get going with Octopress. The next step is to set up a user so that you don't have to run the rake commands as root.

```
# Add blogger user
  RUN adduser --disabled-password --gecos "" blogger && \
  echo "blogger ALL=(root) NOPASSWD:ALL" > /etc/sudoers

  USER blogger
```

Next create the working folder `octopress` and grant permissions to blogger to by changing ownership of the folders where you need to make changes in the future

```
# Directory for the blog files
RUN sudo mkdir /octopress
WORKDIR /octopress

# Set permissions so blogger can install gems
  RUN sudo chown -Rv blogger:blogger /octopress
  RUN sudo chown -Rv blogger:blogger /var/lib/gems
  RUN sudo chown -Rv blogger:blogger /usr/local/bin
```

Running `rake preview` in your octopress blog folder will generate the blog and serve it on port 4000. In order to access the blog from outside the container you need to tell Docker to expose port 4000 for connections.

``` 
# Expose port 4000 so we can preview the blog
EXPOSE 4000
```

Next it adds the `Gemfile`. The contents of this file will be custom to your blog so copy it from the blog repository as mentioned earlier. Easiest is to copy your `Gemfile` to be in the same folder as the `Dockerfile` since the docker `ADD` command is relative to the directory you build from. 

The next section will add the `Gemfile` to the Docker image and install the bundles.
```
  # Add the Gemfile and install the gems
  ADD Gemfile /octopress/Gemfile
  RUN gem install bundler
  RUN bundle install
```

Then it adds your gitconfig to the image. This is necessary to provide the same git experience in or outside of the container. As with the `Gemfile` you'll have to copy your `.gitconfig` file to the same folder as the `Dockerfile`
```
ADD .gitconfig /home/blogger/.gitconfig
```

And that is it. See the repository mentioned above for the complete Dockerfile.

#### Build the docker image

Everything is ready so you can now build the Docker image.

* Get the Dockerfile from Github `git clone git@github.com:dirkvanrensburg/octopress2-dockerfile.git octopress-dockerfile`
* Then copy the `Gemfile` from the blog repository into the same folder. 
* Then copy your `.gitconfig` file from your home folder into the same folder
* Then in the folder containing you `Dockerfile` run the following:

```
docker build . -t blog/octopress
```
Flag  | Description
------|------------
__-t__| Tag the built image with that name. Later you can use the tag to start a container.

<br/>

This command instructs Docker to create a container image using the instructions in the `Dockerfile`. If all goes well you should see a message saying something like this: `Successfully built b847ccd963fa`

#### Test the container

To test the Docker image, start the container using the following command:

```
docker run --rm -ti blog/octopress /bin/bash
```
Flag  | Description
------|------------
__--rm__| Clean up by removing the contaier and its filesystem when it exits.
__-ti__| Tells docker to create a pseudo TTY for the container and to keep the standard in open so you can send keystrokes to the container.

<br/>
The container will start up and your terminal should be attached and in the `octopress` folder. You should see something like:

```
blogger@eba0e6a691ef:/octopress$
```

The `exit` command will exit the container and clean up.

### Rakefile

In order to preview the blog while working on a post, you need to change the `Rakefile` in the root of the blog repository to bind the preview server to the IP wildcard `0.0.0.0`. This makes it possible to access the blog preview in your browser at `http://localhost:4000`

**Change**
```
  rackupPid = Process.spawn("rackup --port #{server_port}")
```

**To**
```
  rackupPid = Process.spawn("rackup -o 0.0.0.0 --port #{server_port}")
```

### Launch the container

The next step is to launch the container. Execute this command from anywhere, replacing the paths to your blog and .ssh keys:

```
docker run -p 4000:4000 --rm --volume <absolute-path-to-blog-repository>:/octopress --volume <absolute-path-to-user-home>/.ssh:/home/blogger/.ssh -ti blog/octopress /bin/bash
```

Flag    | Description
--------|-------------
__docker run__ | Instructs docker to run a previously built image. `blog/octopress` in this case
__-p 4000:4000__ | Instructs docker to expose the internal (to the container) port to the host interfaces so that the host can send data to a process listening on that port in the container
__--rm__ | Tells docker to remove the container and its file system when it exits. We don't need to keep the container around since the blog source is external to the image
__--volume__ | `volume` is used to mount folders on the host system into the container. This allow processes in the container to access the files as if they are local to the container. In this case two folders are mounted. The blog repository as `/octopress` and the local `.ssh` folder of the host user as `/home/blogger/.ssh`. The ssh keys are used by git to authenticate and encrypt traffic to and from github. Feel free to change this so that only the github keys are available in the container. 
__-ti__ | Tells docker to create a pseudo TTY for the container and to keep the standard in open so you can send keystrokes to the container.
__blog/octopress__ | The name of the image to run. This is the image built earlier using `docker build`
__/bin/bash__ | The command to run when starting the container. 

<br/>

Docker will start the container, create a pseudo tty, open standard in and run `/bin/bash` so that your terminal is now effectively _inside_ the container.

> <small>It is handy to place the command above in a script in the `~/bin` folder of your user. For example create a file called `~/bin/blog` and place the command in there. Then you can run `blog` from any terminal to immediately start and access the container. </small>


### Do blog stuff

Now run the Octopress blogging commands as you would if Ruby was installed on your local machine. 

#### Install theme

If you created a new Octopress blog then you now have to install your theme. The following installs the default Octopress theme:

```
rake install
```

#### New post
```
rake new_post
```
Will ask for the post name and create the post file in `source/_posts`

#### Preview

To preview blog posts:
```
rake preview
```

Then in your browser navigate to `http://localhost:4000` and you should see the preview of your blog.

#### Generate
```
rake generate
```
This will generate the blog into the `public` folder

#### Publish
```
rake deploy
```

This will commit the generated blog and push it to github.


[octopress]: http://octopress.org/
[octo-docs]: http://octopress.org/docs/
[docker]: https://www.docker.com/
[dockerfile-docs]: https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices
[rubylang]: https://www.ruby-lang.org/en/
[rbenv]: https://github.com/rbenv/rbenv
[markdown]: https://www.google.com.au/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=markdown
[github-pages]: https://pages.github.com/
