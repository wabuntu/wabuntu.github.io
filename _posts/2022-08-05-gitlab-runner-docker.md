---
layout: post
title: "Gitlab and Runner with Docker"
date: 2022-08-05 00:00:00 +0900
lang: ja
---

## Gitlab and Runner with Docker

### assign 10.10.10.100 for gitlab and launch

```bash
$ sudo docker network create --subnet 10.10.10.0/24 --attachable net-internal
$ sudo docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
e28e2622a90c   bridge           bridge    local
4b43153cf7a0   docker_default   bridge    local
246204e8528d   gitlab_default   bridge    local
0dbf68ab8797   host             host      local
e952003cd965   net-internal     bridge    local
ab764d0c9688   none             null      local
78cfad2c972a   runner_default   bridge    local

$ export GITLAB_HOME=/srv/gitlab

$ sudo docker run --detach \
  --hostname gitlab.example.com \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab:Z \
  --volume $GITLAB_HOME/logs:/var/log/gitlab:Z \
  --volume $GITLAB_HOME/data:/var/opt/gitlab:Z \
  --shm-size 256m \
  --network net-internal \
  --ip 10.10.10.100 \
  gitlab/gitlab-ee:latest
 
$ sudo docker logs -f gitlab
```

### Check root password

```bash
$ sudo docker exec -it gitlab cat /etc/gitlab/initial_root_password
$ sudo docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"
$ curl http://gitlab.example.com
<html><body>You are being <a href="http://127.0.0.1/users/sign_in">redirected</a>.</body></html>
```

### Install runner

```bash
$ sudo docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --network net-internal \
  --ip 10.10.10.101 \
    gitlab/gitlab-runner:latest
```

### Check IP and connection

```bash
$ sudo docker exec -it gitlab ip a
$ sudo docker exec -it gitlab-runner curl http://10.10.10.100/
$ sudo docker exec -it gitlab-runner curl http://gitlab.example.com/
```

Go to https://gitlab.example.com/, create a project, then navigate to CI/CD->Runner in the project.

```bash
$ GITWEB=http://gitlab.example.com/
$ TOKEN=<your-registration-token>

$ sudo docker exec -it gitlab-runner gitlab-runner register --url $GITWEB --registration-token $TOKEN
```

```bash
$ sudo vi /srv/gitlab-runner/config/config.toml
```

```toml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "3newconf"
  url = "http://gitlab.example.com/"
  token = "XjD5kW5tVpLMQxF_4EEw"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "opensuse/leap"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
    extra_hosts = ["gitlab.example.com:10.10.10.100"]
    network_mode = "host"
    volumes = ["/etc/hosts:/etc/hosts:ro"]
```

```bash
$ docker restart gitlab-runner
```

### How to let runner to understand gitlab url

All options for docker daemon: https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file

All runner options: https://docs.gitlab.com/runner/configuration/advanced-configuration.html

### When it fails to start after reboot

```bash
$ sudo docker ps -a
CONTAINER ID   IMAGE                         COMMAND                  CREATED      STATUS                      PORTS     NAMES
001fe541e4c0   gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   3 days ago   Exited (128) 15 hours ago             gitlab-runner
5542b77d64f4   gitlab/gitlab-ee:latest       "/assets/wrapper"        3 days ago   Exited (137) 15 hours ago             gitlab

$ sudo docker start gitlab
Error response from daemon: failed to create endpoint gitlab on network net-internal: network d1d6e6dd does not exist
Error: failed to start containers: gitlab

$ sudo docker network ls
721b5dea1fc4   net-internal     bridge    local

$ sudo docker network rm net-internal
$ sudo docker network create --subnet 10.10.10.0/24 --attachable net-internal
$ sudo docker network connect net-internal gitlab-runner
$ sudo docker network connect net-internal gitlab
$ sudo systemctl stop ssh
$ sudo docker start gitlab
$ sudo docker start gitlab-runner

$ sudo docker exec -it gitlab ip a
    inet 10.10.10.2/24 brd 10.10.10.255 scope global eth0

$ sudo cat /srv/gitlab-runner/config/config.toml

  [runners.docker]
    volumes = ["/cache", "/etc/hosts:/etc/hosts:ro"]
    extra_hosts = ["gitlab.example.com:10.10.10.2"]
    network_mode = "host"
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2022/08/05/gitlab-and-runner-with-docker/).*
