# Docker User Namespaces Enforcement Plugin

This project provides a simple Docker authorization plugin that prevents the running of containers with userns-mode set to host (`--userns=host`) when [Docker user namespace remapping](https://docs.docker.com/engine/security/userns-remap/) is enabled.

## Background
- ["Privilege escalation" when starting the Docker daemon with user namespaces enabled](https://github.com/moby/moby/issues/32624)
- [Docker user namespaces](https://docs.docker.com/engine/security/userns-remap/)
- [Docker daemon attack surface](https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface)

The use of "Docker user namespaces" is **not** a solution to the widely known "*anyone with access to the Docker daemon effectively has root permissions on the host*" problem.

**This is because users can disable Docker user namespaces for any container by adding the `--userns=host` flag to the `docker container create`, `docker container run`, or `docker container exec` commands.** [Reference](https://docs.docker.com/engine/security/userns-remap/#disable-namespace-remapping-for-a-container)

```bash
[testuser@fedora31 ~]$ id
uid=1001(testuser) gid=1001(testuser) groups=1001(testuser),974(docker)
[testuser@fedora31 ~]$ grep testuser /etc/sub?id
/etc/subgid:testuser:165536:65536
/etc/subuid:testuser:165536:65536
[testuser@fedora31 ~]$ cat /etc/docker/daemon.json 
{
    "userns-remap": "testuser"
}
[testuser@fedora31 ~]$ docker run -it --rm --volume="/:/mnt/hostfs" fedora /bin/bash
[root@1d135f34d711 /]# cd /mnt/hostfs/root/
bash: cd: /mnt/hostfs/root/: Permission denied
...
testuser@fedora31 ~]$ docker run -it --rm --volume="/:/mnt/hostfs" --userns=host fedora /bin/bash
[root@b6441243f429 /]# cd /mnt/hostfs/root/
[root@b6441243f429 root]# 
```

## Installation and configuration of the plugin

