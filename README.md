# README
I try to document my Docker knowledge in one place and add pointers/remeber/spoilers so I do not have to reread all documentation.



More sources I should read:

* https://runnable.com/docker/getting-started/
* https://www.javatpoint.com/what-is-docker-volume#:~:text=Docker%20volumes%20are%20file%20systems,if%20another%20process%20needs%20it.



# CLI cheatsheet

* `docker-compose stop` is not `docker-compose down`
  * `docker-compose stop` only stops the container but does not remove stuff
  * `docker-compose down` stops the container and removes everything except for the volumes.

* `docker-compose start` will not create containers and pull images
* `docker-compose up` will not create containers and pull images
* `docker run` : creates a new container from the referenced Docker image

`docker run -d --name test1 dumlutimuralp/networktest` (this would start the container in background)

`docker exec -it < container id > bash` (This would attach the terminal to the container that is running in the background. To execute commands in the container)

`docker run -it --name test1 dumlutimuralp/networktest /bin/bash` (This command would start the container in the foreground and attach the terminal to the container right away. To properly exit, without killing the container, use ctrl + P + Q)

* `docker pull` : copies images to docker host
* `docker images` : lists images on the docker host
* `docker rmi` : removes images from the docker host
* `docker ps` : lists the running containers