# Docker Volume Backups Explained

[toc]

When I was looking for a way to back up Docker named volumes I was surprised to find out that there’s no standard way of handling the process. In the [official documentation](https://docs.docker.com/engine/tutorials/dockervolumes/#backup-restore-or-migrate-data-volumes) there’s only a note about using data volume containers and a `--volumes-from` option. There’s also a [docker ](https://docs.docker.com/engine/reference/commandline/cp/)`cp`[ command](https://docs.docker.com/engine/reference/commandline/cp/), but it requires the volume path inside the container that uses them, which makes it less generic.

After a bit of research, it turned out it’s actually pretty easy to back up volumes using volume mounts and a `tar` utility. For example to backup `some_volume` to `/tmp/some_archive.tar.bz2`, you can simply run:

```
docker run --rm -v some_volume:/volume -v /tmp:/backup alpine tar -cjf /backup/some_archive.tar.bz2 -C /volume ./
```

And to restore it, just run:

```
docker run --rm -v some_volume:/volume -v /tmp:/backup alpine sh -c "rm -rf /volume/* /volume/..?* /volume/.[!.]* ; tar -C /volume/ -xjf /backup/some_archive.tar.bz2"
```

**Note**: As a safety precaution, I advise you to stop all the containers using the volume being backed-up or restored, otherwise an inconsistent / intermediate state might be archived or recovered.

In the end I wrote my own little [volume-backup](https://github.com/loomchild/volume-backup) utility that simplifies the backup & restore process even further and offers some additional benefits, such as using `stdout` and `stdin` streams.

[loomchild/volume-backupdocker volume backup & restore utility. Contribute to loomchild/volume-backup development by creating an account on…github.com](https://github.com/loomchild/volume-backup)

Here’s the example backup command:

```
docker run --rm -v some_volume:/volume -v /tmp:/backup loomchild/volume-backup backup some_archive
```

And here’s an example restore command:

```
docker run --rm -v some_volume:/volume -v /tmp:/backup loomchild/volume-backup restore some_archive
```

Feel free to check out the [project page on GitHub](https://github.com/loomchild/volume-backup) for complete documentation and let me know what you think.

*Edit: Changed the cleanup code to delete hidden files — thanks Olivier.*

*Edit: It’s also possible to backup to standard output and restore from standard input. I added this capability to* [*volume-backup*](https://github.com/loomchild/volume-backup#backup-to-standard-output) *— thanks Holger.*

*Edit: Added –rm flag to remove the container when finished — thanks awade.*