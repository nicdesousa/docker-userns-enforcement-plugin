# Docker User Namespaces Enforcement Plugin

![Go Report Card](https://goreportcard.com/badge/github.com/nicdesousa/docker-userns-enforcement-plugin)
![GitHub](https://img.shields.io/github/license/nicdesousa/docker-userns-enforcement-plugin)
[![CodeFactor](https://www.codefactor.io/repository/github/nicdesousa/docker-userns-enforcement-plugin/badge)](https://www.codefactor.io/repository/github/nicdesousa/docker-userns-enforcement-plugin)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=nicdesousa_docker-userns-enforcement-plugin&metric=alert_status)](https://sonarcloud.io/dashboard?id=nicdesousa_docker-userns-enforcement-plugin)

This project provides a simple Docker authorization plugin that prevents the running of containers with userns-mode set to host (`--userns=host`) when [Docker user namespace remapping](https://docs.docker.com/engine/security/userns-remap/) is enabled.

## Background
- ["Privilege escalation" when starting the Docker daemon with user namespaces enabled](https://github.com/moby/moby/issues/32624)
- [Docker user namespaces](https://docs.docker.com/engine/security/userns-remap/)
- [Docker daemon attack surface](https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface)

The use of "Docker user namespaces" is **not** a solution to the widely known "*anyone with access to the Docker daemon effectively has root permissions on the host*" problem.

**This is because users can disable Docker user namespaces for any container by adding the `--userns=host` flag to the `docker container create`, `docker container run`, or `docker container exec` commands.** [See the Docker documentation](https://docs.docker.com/engine/security/userns-remap/#disable-namespace-remapping-for-a-container)

```bash
$ id
uid=1001(testuser) gid=1001(testuser) groups=1001(testuser),974(docker)
$ grep testuser /etc/sub?id
/etc/subgid:testuser:165536:65536
/etc/subuid:testuser:165536:65536
$ cat /etc/docker/daemon.json 
{
    "userns-remap": "testuser"
}
...
# Daemon-configured (default) user namespace:
$ docker run -it --rm --volume="/:/mnt/hostfs" fedora /bin/bash
[root@1d135f34d711 /]# cd /mnt/hostfs/root/
bash: cd: /mnt/hostfs/root/: Permission denied
...
# User-specified user namespace:
$ docker run -it --rm --volume="/:/mnt/hostfs" --userns=host fedora /bin/bash
[root@b6441243f429 /]# cd /mnt/hostfs/root/
[root@b6441243f429 root]# 
...
# Solution provided by this plugin:
$ docker run -it --rm --volume="/:/mnt/hostfs" --userns=host fedora /bin/bash
docker: Error response from daemon: authorization denied by plugin deny-userns-mode-host: userns=host is not allowed.
See 'docker run --help'.
```

## Installation and configuration of the plugin

1. Download and compile the plugin source code:
```bash
$ go get github.com/nicdesousa/docker-userns-enforcement-plugin
```
2. Copy the `docker-userns-enforcement-plugin` binary to a suitable directory:
```bash
$ sudo cp $GOPATH/bin/docker-userns-enforcement-plugin /usr/local/bin
```
3. Create a systemd service and socket file for the plugin:

`/etc/systemd/system/docker-userns-enforcement-plugin.service`
```bash
[Unit]
Description=Docker User Namespaces Enforcement Plugin
Before=docker.service
After=network.target docker-userns-enforcement-plugin.socket
Requires=docker-userns-enforcement-plugin.socket docker.service

[Service]
ExecStart=/usr/local/bin/docker-userns-enforcement-plugin

[Install]
WantedBy=multi-user.target
```
`/lib/systemd/system/docker-userns-enforcement-plugin.socket`
```bash
[Unit]
Description=Docker User Namespaces Enforcement Plugin

[Socket]
ListenStream=/run/docker/plugins/deny-userns-mode-host.sock

[Install]
WantedBy=sockets.target
```

Enable the plugin:
```bash
# systemctl daemon-reload
# systemctl enable --now docker-userns-enforcement-plugin
```
4. Configure the Docker daemon to use the plugin:

`/etc/docker/daemon.json`
```bash
{
    "authorization-plugins": ["deny-userns-mode-host"],
    "userns-remap": "testuser"
}
```

Restart the docker service:
```bash
# systemctl restart docker
```

5. Test with the examples provided in the Background section.
