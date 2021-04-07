# Docker Cronjobs explained

## Option 1

You are way better off running Docker cron jobs from your host system, not from inside your Docker containers. Your Docker containers should only have one concern only, and not be saddled with the weight of cronjobs.

Use this instead on your host system’s crontab:

```
* * * * * docker run --rm your-container /some/cool/task.sh
```

This is what I use on the mo5t:
```
45 23 * * * /home/paulb/.docker/scripts/mo5tbackup2st1.sh
```
With a bash script that dubs Postgres and MySQL like this:
```bash
docker-compose exec -T -u postgres postgres pg_dump -U netbox netbox > $DOCKERDIR/dbdump/netbox.sql 2> $LOG

docker-compose exec -T db mysqldump --password=xxxxxx --user=root --add-drop-database --all-databases | grep -v "Using a password" > $DOCKERDIR/dbdump/tilt.sql 2> $LOG
```

## Option 2

In the same folder as your Dockerfile, create a file called `crontab`:

```
* * * * * /some/cool/task.sh
# Mandatory blank line
```

Then add the following to your Dockerfile:

```
COPY crontab /etc/cron.d/cool-task
RUN chmod 0644 /etc/cron.d/cool-task
RUN service cron start
```

Rebuild the Docker image, and you’re all set!