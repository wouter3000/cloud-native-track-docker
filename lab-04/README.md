# Lab 04 - Dockerfile best-practices

The goal when building your own Docker images is to keep them as small as
possible.  The main reason is of course that the smaller the image, the faster
it can be distributed (pulled/pushed).  Another reason why it is important to
keep an image as small/minimal as possible is that the fewer packages are
installed in the image, the smaller the attack surface of the image (less
possible vulnerabilities).

## Task 1: Looking for the right base image (FROM)

Base images differ greatly in size, so it is important to look at your
requirements when choosing a base image.

```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              latest              5cb3aa00f899        11 hours ago        5.53MB
busybox             latest              d8233ab899d4        3 weeks ago         1.2MB
ubuntu              latest              47b19964fb50        4 weeks ago         88.1MB
centos              latest              1e1148e4cc2c        3 months ago        202MB
```

Because of its small size, the alpine image is used as base image for a lot of
the official images.  But which base image you should use totally depends on
your requirements and use-case.  Often specific base images (for example RHEL)
have to be used within companies.

It has already been mentioned before, but it recommened to always use a specific
version (tag) of the base image (and not `latest`).  Not doing this will prevent
you from creating reproduceable images.  For example:

```
FROM centos:7.6.1810
```

## Task 2: Using labels (LABEL)

Labels are often forgotten, but they can be very handy to set some metadata on
your image.  Most commonly there is a label for the maintainer of the image, but
you can basically add whatever metadata you want using labels.

```
FROM centos:7.6.1810

LABEL maintainer="steven.trescinski@gluo.be"
LABEL licence="GPL"
```

As seen in the previous lab you can use the `docker inspect` command to fetch
the labels of an image.

## Task 3: Running commands (RUN)

If you want to run commands (to install packages for example) in the
container/image we will need to use `RUN` in our Dockerfile.  Important to know
is that every `RUN` statement in your Dockerfile will result in an additional
layer in your Docker image.  So it is best practice to work as few `RUN`
statements as possible in your Dockerfile.

What you do not want to do is the following:

```
FROM centos:7.6.1810

LABEL maintainer="steven.trescinski@gluo.be"
LABEL licence="GPL"

RUN yum -y update
RUN yum -y install elinks
RUN yum -y install wget
RUN yum clean all
```

Create a `Dockerfile` with the above content and build and image from it:

```
docker build -t centos_multiple_run:v1 .
```

Now we will do the same, but this case with everything stringed into a single
`RUN` statement. Edit your existing `Dockerfile` so it looks like this:

```
FROM centos:7.6.1810

LABEL maintainer="steven.trescinski@gluo.be"
LABEL licence="GPL"

RUN yum -y update && \
    yum -y install elinks wget && \
    yum clean all
```

Build a new image:

```
docker build -t centos_single_run:v1 .
```

When we compare the two images we can see that there is more than 200MB
difference between these two images:

```
REPOSITORY            TAG                 IMAGE ID            CREATED              SIZE
centos_single_run     v1                  71df038da8e4        5 seconds ago        303MB
centos_multiple_run   v1                  7c7285fd7969        About a minute ago   536MB
```

## Task 4: Optimally using the build cache

As already mentioned, each line in a Dockerfile generates a new layer in the
resulting image.  Docker tries to be as efficient as possible when building
images.  So whenever possible it tries to cache as many layers as possible, but
as soon as one layer cannot be used from the cache all next layers will not be
used from the cache either.  This could lead to quite some differences in build
times.

Create a Dockerfile with the following content:

```
FROM centos:7.6.1810

LABEL maintainer="steven.trescinski@gluo.be"
LABEL licence="GPL"

ARG hello_world_string

RUN touch /hello-world.txt && \
    echo "${hello_world_string}" > /hello-world.txt

RUN for i in {0..30}; do echo "$i"; sleep 1; done; echo

CMD ["cat", "/hello-world.txt"]
```

Build two images from the above Dockerfile (we are using different variables):

```
docker build --build-arg hello_world_string='Hello world 1!' -t centos_hello_world_1 .
docker build --build-arg hello_world_string='Hello world 2!' -t centos_hello_world_2 .
```

By simply switching the order of the two `RUN` lines and the `ARG` line we can
tremendously speed up the builds:

```
FROM centos:7.6.1810

LABEL maintainer="steven.trescinski@gluo.be"
LABEL licence="GPL"

RUN for i in {0..30}; do echo "$i"; sleep 1; done; echo

ARG hello_world_string

RUN touch /hello-world.txt && \
    echo "${hello_world_string}" > /hello-world.txt

CMD ["cat", "/hello-world.txt"]
```

Build two new images from the new Dockerfile (we are again using different
variables):

```
docker build --build-arg hello_world_string='Hello world 3!' -t centos_hello_world_3 .
docker build --build-arg hello_world_string='Hello world 4!' -t centos_hello_world_4 .
```

Because we are making changes to the `RUN` statement referencing the
`${hello_world_string}` variable all the lines below it will not be taken from
cache and will slow the build.  The initial build is still slow (you will see
the counter), but the build  after that will be a lot quicker as the cache is
used more optimally!

## Task 5: Docker build User

A simple example to show how the `USER` tag works in a `Dockerfile` we will create
a default `Dockerfile` without a `USER` tag. This `Dockerfile` is going to be
very simple.

```
FROM nginx
```

So basically we are just creating a new `nginx` image with the base `nginx` image.
It's not something that should be done but this is for workshop purposes only.

So if the image is build we will see that we connect to the container with the
`root` user.

```
docker build -t my_nginx .

Sending build context to Docker daemon  2.048kB
Step 1/1 : FROM nginx
 ---> 53f3fd8007f7
Successfully built 53f3fd8007f7
Successfully tagged my_nginx:latest
```

Now we will execute into the container.

```
docker run -ti my_nginx bash

root@6ae438732ee2:/#
```

Now build the image with the `USER` tag in the `Dockerfile`.

```
FROM nginx

USER nginx
```

Again, this is a very simple `Dockerfile` but it does exactly what we are trying
to achieve in this task.

Build the image again with the same `Dockerfile`.

```
docker build -t my_nginx .

Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 53f3fd8007f7
Step 2/2 : USER nginx
 ---> Running in 47f7fa86101d
Removing intermediate container 47f7fa86101d
 ---> 75ef857f7112
Successfully built 75ef857f7112
Successfully tagged my_nginx:latest
```

If there is going to be executed into the container now it will connect with the
`nginx` user.

```
docker run -ti my_nginx bash

nginx@1751e41489bb:/$
```

## Task 6: Entrypoints vs. CMD  

In a nutshel, `CMD` sets default command and/or parameters, which can be overwritten
from command line when docker container runs.

`ENTRYPOINT` configures a container that will run as an executable.

So let's start with creating an image with an `ENTRYPOINT` that does `echo Hello
World!`.

```
FROM nginx

ENTRYPOINT ["/bin/echo" , "Hello World!"]
```

Build the `hello-world` image with this `Dockerfile`.

```
docker build -t hello-world .

Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 53f3fd8007f7
Step 2/2 : ENTRYPOINT ["/bin/echo" , "Hello World!"]
 ---> Running in 50c122fdb47c
Removing intermediate container 50c122fdb47c
 ---> 44314b2c9cc9
Successfully built 44314b2c9cc9
Successfully tagged hello-world:latest
```

The image is now created with an `ENTRYPOINT` this means that we won't be able to
overwrite the initial `command` the container is going to do. It is possible to
add more arguments to this command though.

```
docker run -ti hello-world

Hello World!

docker run -ti hello-world my argument

Hello World! my argument
```

So this means that if you are going to do `-ti bash` you are not actually
overwriting the `echo` command but you are just adding the `bash` string to the
`echo` command.

```
docker run -ti hello-world bash

Hello World! bash
```

This is totally different from the `CMD` line in the `Dockerfile`. Create the
image now with the `CMD` line.

```
FROM nginx

CMD ["/bin/echo" , "Hello World!"]
```

Build the image with the edited `Dockerfile`.

```
docker build -t hello-world .

Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 53f3fd8007f7
Step 2/2 : CMD ["/bin/echo" , "Hello World!"]
 ---> Running in 042aca86ca67
Removing intermediate container 042aca86ca67
 ---> a2ec1c7081a6
Successfully built a2ec1c7081a6
```

If you will run the image now you will see that you get the same output as the
`ENTRYPOINT` line. This is because we are currently not overwriting the command. 

```
docker run -ti hello-world

Hello World!
```

If we are going to add `bash` to the `run` command you will see that we will
`exec` into the container, thus overwrite the initial command.

```
docker run -ti hello-world bash

root@fa9a3a2a62f1:/#
```

## Task 7: Multi-stage image builds

Another way to keep Docker images as small as possible is to use the multi-stage
functionality.  With it you can use different images at different stages, this
is useful when you build you artefact in the first stage (where you will require
all the different build tools) and deploy that artefact in into a lean and clean
image.

To show the power of multi-stage image builds we are going to use an Golang
example.  Create a file `main.go` in your working directory with the following
content:

```
package main

import "fmt"

func main() {
	fmt.Println("Hello world")
}
```

We will first build it without using multi-stage image builds.  To do so create
a Dockerfile with the following content:

```
FROM golang:1.8.3
WORKDIR /go/src/hello-world
COPY main.go /go/src/hello-world
RUN go install
CMD ["hello-world"]
```

And build the image:

```
docker build -t golang-hello-world:legacy .
```

Run the image to see that it works:

```
docker run golang-hello-world:legacy
```

Now change the Dockerfile content to:

```
FROM golang:1.8.3 as builder
WORKDIR /go/src/hello-world
COPY main.go ./
RUN go install
RUN ldd /go/bin/hello-world | grep -q "not a dynamic executable"

FROM scratch
COPY --from=builder /go/bin/hello-world /hello-world
CMD ["/hello-world"]
```

And build the image again:

```
docker build -t golang-hello-world:multistage .
```

Run the image to see that it works:

```
docker run golang-hello-world:multistage
```

Check the size differce of the images:

```
REPOSITORY             TAG                 IMAGE ID            CREATED              SIZE
golang-hello-world     multistage          ab62167f4c3d        19 seconds ago       1.55MB
golang-hello-world     legacy              d1c7e682e0fa        About a minute ago   701MB
```

The difference is huge, a couple of MB's instead of almost 1 GB.  For JAVA we
can of course do the same.  Using a JDK image for the build process and a JRE
image for the run process.

## Task 6: clean up

To clean up everything run the following commands:

```
docker system prune
```
