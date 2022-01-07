# Dockerhub Images Explained

[toc]



# TL;DR

* Repo hub https://hub.docker.com/
* Tutorial https://www.youtube.com/watch?v=iqqDU2crIEQ&t=1002s
* See Docker file examples: https://github.com/paulboot/gns3-registry/tree/master/docker



The easiest way to package your code for production is by using a container image. DokerHub is like Github for container images— you can upload and share with others an unlimited amount of publicly available dockerized applications at no cost. In this article, we’ll build a simple image and push it to Dockerhub.

# 1. Sign up for a free Dockerhub account

The Docker ID you choose is important as you will need to tag all your images with it. For instance, if your Docker ID is **ml-practitioner**, then all your images will have to be tagged in the form:

```
ml-practitioner/image-name:latest
```

If you want to create several versions of your container image, you can tag them with different versions, for instance:

```
mlpractitioner/mydatascienceimage:1.2
```

# 2. Create a Dockerfile

A `Dockerfile` contains a set of instructions for your Docker image. You can think of it as a recipe for a cake. Once you build and share your image on Dockerhub, it’s like sharing your curated recipe with others on your cooking blog. Anyone can then take this well-prepared recipe (*your Dockerhub image*) and either directly bake a cake based on that (*running a container instance from that image as-is*), or make some modifications and create even fancier recipes (*use it for other Docker images*).

In fact, in the simple Dockerfile example below, we’re doing just that: we’re using an image `python:3.8` as a base image (*base recipe*) and we’re creating our own image out of it, which has the effect of adding new layers to the base image:

```
# set a base image: https://hub.docker.com/_/python?tab=tags&page=1&ordering=last_updated
FROM python:3.8

# optional: ensure that pip is up to date
RUN pip install --upgrade pip

# optional: environment
ENV PORT 80
ENV HOSTNAME=gns3vm

# optional: expose ports
EXPOSE 8888

# first we COPY only requirements.txt to ensure that later builds
# with changes to your src code will be faster due to caching of this layer
COPY requirements.txt .
RUN pip install -r requirements.txt

# copy all your custom modules and files from the src directory
# NOTE .dockerignore can be used to ignore files!
COPY src/ .

# specify the script that will be executed on container start
CMD [ "python", "etl.py"]
```

The `COPY` command copies your files to the container directory, and `CMD` determines the code that is executed when the container is started via `docker run image-name`.

# 3. Build your Docker image

Let’s say, my image will be called `etlexample`. In our terminal, we switch to the directory, where we have our Dockerfile, requirements.txt, and `src` directory with our main code (*can contain several modules*).

Let’s build a simple ETL example. Here is a project structure that we will use:

```
|-- Dockerfile
|-- requirements.txt
 -- src/
    -- etl.py
```

Your `requirements.txt` could look as follows (*just an example*):

```
pandas==1.2
scikit-learn==0.24
```

We then need to **build our image** and -t (== --tag list)

```
docker build -t etlexample .
```

The dot at the end indicates a build context — for us, it means that we use `Dockerfile` from the current working directory for this build.

Now your image is ready! When you type `docker image ls`, you should be able to see your image.

# 4. Run image

If you do not name your image docker will give it a random unique`name` it. that is NOT a `tag` because the same `tag` can run multiple times with unique `names`.

`docker run etlexample`

or expose a port

`docket run -p 8080:80 --name hello -d hello-world`

check it it is running

`docker ps -a`

check and tail the logs

`docker logs -f`

and stop it

`docker stop <NAME>`

# 4. Log in to Dockerhub from your Terminal

```
docker login -u myusername
```

`myusername` indicates the Docker ID that we chose when signing up. You will be then prompted to type your password. Then, you should see: `Login succeeded`.

# 5. Tag your image

By default, when we build an image, it assigns the tag `1.0.0` and not the confusing tag `latest` simply means the “default version” of your image. When tagging, you could assign some specific version to it. For now, we’ll go with the `latest` tag.

```
docker image tag etlexample:1.0.0 myusername/etlexample:1.0.0
```

When you now type `docker image ls` again, you will see the image with your Dockerhub username.

```
See images and check tag and same IMAGE ID!
<ToDo>
```



# 6. Push your image to Dockerhub

```
docker image push myusername/etlexample:1.0.0
```

# 7. Test and confirm a success

When you now try to pull this image, you should see:

```
docker pull myusername/etlexample:1.0.0

```

