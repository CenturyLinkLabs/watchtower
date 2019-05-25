<p align="center">
  <img src="https://0x12b.com/watchtower-logo.png" width="450" />
</p>
<h1 align="center">
  Watchtower
</h1>

<p align="center">
  A process for automating Docker container base image updates.
  <br/><br/>
  <a href="https://circleci.com/gh/containrrr/watchtower">
    <img alt="Circle CI" src="https://circleci.com/gh/containrrr/watchtower.svg?style=shield" />
  </a>
  <a href="https://godoc.org/github.com/containrrr/watchtower">
    <img alt="GoDoc" src="https://godoc.org/github.com/containrrr/watchtower?status.svg" />
  </a>
  <a href="https://microbadger.com/images/containrrr/watchtower">
    <img alt="Microbadger" src="https://images.microbadger.com/badges/image/containrrr/watchtower.svg" />
  </a>
  <a href="https://goreportcard.com/report/github.com/containrrr/watchtower">
    <img alt="Go Report Card" src="https://goreportcard.com/badge/github.com/containrrr/watchtower" />
  </a>
  <a href="https://github.com/containrrr/watchtower/releases">
    <img alt="latest version" src="https://img.shields.io/github/tag/containrrr/watchtower.svg" />
  </a>
  <a href="https://www.apache.org/licenses/LICENSE-2.0">
    <img alt="Apache-2.0 License" src="https://img.shields.io/github/license/containrrr/watchtower.svg" />
  </a>
  <a href="https://www.codacy.com/app/simskij/watchtower">
    <img alt="Codacy Badge" src="https://api.codacy.com/project/badge/Grade/3a4d0fcfd26d45b09b1d7ea3c8c13744"/>
  </a>
  <a href="https://www.codacy.com/app/simskij/watchtower?utm_source=github.com&utm_medium=referral&utm_content=containrrr/watchtower&utm_campaign=Badge_Coverage">
    <img alt="Codacy Badge" src="https://api.codacy.com/project/badge/Coverage/3a4d0fcfd26d45b09b1d7ea3c8c13744" />
  </a>
  <a href="https://gitter.im/containrrr/watchtower?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge">
    <img alt="Join the chat at https://gitter.im/containrrr/watchtower" src="https://badges.gitter.im/containrrr/watchtower.svg" />
  </a>
  <a href="#contributors">
    <img alt="All Contributors" src="https://img.shields.io/badge/all_contributors-30-orange.svg?style=flat-square" />
  </a>
</p>

## Overview

Watchtower is an application that will monitor your running Docker containers and watch for changes to the images that those containers were originally started from. If watchtower detects that an image has changed, it will automatically restart the container using the new image.

With watchtower you can update the running version of your containerized app simply by pushing a new image to the Docker Hub or your own image registry. Watchtower will pull down your new image, gracefully shut down your existing container and restart it with the same options that were used when it was deployed initially.

For example, let's say you were running watchtower along with an instance of _centurylink/wetty-cli_ image:

```bash
$ docker ps
CONTAINER ID   IMAGE                   STATUS          PORTS                    NAMES
967848166a45   centurylink/wetty-cli   Up 10 minutes   0.0.0.0:8080->3000/tcp   wetty
6cc4d2a9d1a5   containrrr/watchtower   Up 15 minutes                            watchtower
```

Every few minutes watchtower will pull the latest _centurylink/wetty-cli_ image and compare it to the one that was used to run the "wetty" container. If it sees that the image has changed it will stop/remove the "wetty" container and then restart it using the new image and the same `docker run` options that were used to start the container initially (in this case, that would include the `-p 8080:3000` port mapping).

## Usage

Watchtower is itself packaged as a Docker container so installation is as simple as pulling the `containrrr/watchtower` image. If you are using ARM based architecture, pull the appropriate `containrrr/watchtower:armhf-<tag>` image from the [containrrr Docker Hub](https://hub.docker.com/r/containrrr/watchtower/tags/).

Since the watchtower code needs to interact with the Docker API in order to monitor the running containers, you need to mount _/var/run/docker.sock_ into the container with the -v flag when you run it.

Run the `watchtower` container with the following command:

```bash
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower
```

If pulling images from private Docker registries, supply registry authentication credentials with the environment variables `REPO_USER` and `REPO_PASS`
or by mounting the host's docker config file into the container (at the root of the container filesystem `/`).

Passing environment variables:

```bash
docker run -d \
  --name watchtower \
  -e REPO_USER=username \
  -e REPO_PASS=password \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower container_to_watch --debug
```

Also check out [this Stack Overflow answer](https://stackoverflow.com/a/30494145/7872793) for more options on how to pass environment variables.

Mounting the host's docker config file:

```bash
docker run -d \
  --name watchtower \
  -v /home/<user>/.docker/config.json:/config.json \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower container_to_watch --debug
```

If you mount the config file as described above, be sure to also prepend the url for the registry when starting up your watched image (you can omit the https://). Here is a complete docker-compose.yml file that starts up a docker container from a private repo at dockerhub and monitors it with watchtower. Note the command argument changing the interval to 30s rather than the default 5 minutes.

```json
version: "3"
services:
  cavo:
    image: index.docker.io/<org>/<image>:<tag>
    ports:
      - "443:3443"
      - "80:3080"
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.docker/config.json:/config.json
    command: --interval 30
```

### Arguments

By default, watchtower will monitor all containers running within the Docker daemon to which it is pointed (in most cases this will be the local Docker daemon, but you can override it with the `--host` option described in the next section). However, you can restrict watchtower to monitoring a subset of the running containers by specifying the container names as arguments when launching watchtower.

```bash
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower nginx redis
```

In the example above, watchtower will only monitor the containers named "nginx" and "redis" for updates -- all of the other running containers will be ignored.

If you do not want watchtower to run as a daemon you can pass a run-once flag and remove the watchtower container after it's execution.

```bash
docker run --rm \
-v /var/run/docker.sock:/var/run/docker.sock \
containrrr/watchtower --run-once nginx redis
```

In the example above, watchtower will execute an upgrade attempt on the containers named "nginx" and "redis". Using this mode will enable debugging output showing all actions performed as usage is intended for interactive users. Once the attempt is completed, the container will exit and remove itself due to the "--rm" flag.

When no arguments are specified, watchtower will monitor all running containers.

### Options

Any of the options described below can be passed to the watchtower process by setting them after the image name in the `docker run` string:

```bash
docker run --rm containrrr/watchtower --help
```

HELP
Type: -
Command Line: --help
Environment Variable: -
Default: -
Show documentation about the supported flags


CLEANUP
Type: --cleanup
Command Line: Boolean
Environment Variable: WATCHTOWER_CLEANUP
Default: False
Remove old images after updating. When this flag is specified, watchtower will remove the old image after restarting a container with a new image. Use this option to prevent the accumulation of orphaned images on your system as containers are updated.

DEBUG
Type: Boolean
Command Line: --debug
Environment Variable: -
Default: False
enable debug mode with verbose logging

HOST
Type: String
Command Line: --host, -h
Environment Variable: DOCKER_HOST
Default: "unix:///var/run/docker.sock"
Docker daemon socket to connect to. Can be pointed at a remote Docker host by specifying a TCP endpoint as "tcp://hostname:port".

INCLUDE STOPPED
Type: Boolean
Command Line: --include-stopped
Environment Variable: WATCHTOWER_INCLUDE_STOPPED
Default:False
Will also include created and exited containers.

INTERVAL
Type: Integer
Command Line: --interval, -i
Environment Variable: WATCHTOWER_POLL_INTERVAL
Default: 300
Poll interval (in seconds). This value controls how frequently watchtower will poll for new images.

LABEL ENABEL
Type: Boolean
Command Line: --label-enable
Environment Variable: WATCHTOWER_LABEL_ENABLE
Default: False
Watch containers where the `com.centurylinklabs.watchtower.enable` label is set to true.

MONITOR ONLY
Type: Boolean
Command Line: --monitor-only
Environment Variable: WATCHTOWER_MONITOR_ONLY
Default: False
Will only monitor for new images, not update the containers.

NO PULL
Type: Boolean
Command Line: --no-pull
Environment Variable: WATCHTOWER_NO_PULL
Default: False
Do not pull new images. When this flag is specified, watchtower will not attempt to pull new images from the registry. Instead it will only monitor the local image cache for changes. Use this option if you are building new images directly on the Docker host without pushing them to a registry.

RUN ONCE
Type: Boolean
Command Line: 
Environment Variable: WATCHTOWER_RUN_ONCE
Default: False
Run an update attempt against a container name list one time immediately and exit.

SCHEDULE
Type: String
Command Line: --schedule, -s
Environment Variable: WATCHTOWER_SCHEDULE
Default: -
[Cron expression](https://godoc.org/github.com/robfig/cron#hdr-CRON_Expression_Format) in 6 fields (rather than the traditional 5) which defines when and how often to check for new images. Either `--interval` or the schedule expression could be defined, but not both. An example: `--schedule "0 0 4 * * *"`

STOP TIMEOUT
Type: Duration
Command Line: --stop-timeout
Environment Variable: WATCHTOWER_TIMEOUT
Default: 10s
Timeout before the container is forcefully stopped. When set, this option will change the default (`10s`) wait time to the given value. An example: `--stop-timeout 30s` will set the timeout to 30 seconds.

TLS VERIFY
Type: Boolean
Command Line: --tlsverify
Environment Variable: DOCKER_TLS_VERIFY
Default: False
Use TLS when connecting to the Docker socket and verify the server's certificate.


See below for options used to configure notifications.

## Linked Containers

Watchtower will detect if there are links between any of the running containers and ensure that things are stopped/started in a way that won't break any of the links. If an update is detected for one of the dependencies in a group of linked containers, watchtower will stop and start all of the containers in the correct order so that the application comes back up correctly.

For example, imagine you were running a _mysql_ container and a _wordpress_ container which had been linked to the _mysql_ container. If watchtower were to detect that the _mysql_ container required an update, it would first shut down the linked _wordpress_ container followed by the _mysql_ container. When restarting the containers it would handle _mysql_ first and then _wordpress_ to ensure that the link continued to work.

## Stopping Containers

When watchtower detects that a running container needs to be updated it will stop the container by sending it a SIGTERM signal.
If your container should be shutdown with a different signal you can communicate this to watchtower by setting a label named _com.centurylinklabs.watchtower.stop-signal_ with the value of the desired signal.

This label can be coded directly into your image by using the `LABEL` instruction in your Dockerfile:

```docker
LABEL com.centurylinklabs.watchtower.stop-signal="SIGHUP"
```

Or, it can be specified as part of the `docker run` command line:

```bash
docker run -d --label=com.centurylinklabs.watchtower.stop-signal=SIGHUP someimage
```

## Selectively Watching Containers

By default, watchtower will watch all containers. However, sometimes only some containers should be updated.

If you need to exclude some containers, set the _com.centurylinklabs.watchtower.enable_ label to `false`.

```docker
LABEL com.centurylinklabs.watchtower.enable="false"
```

Or, it can be specified as part of the `docker run` command line:

```bash
docker run -d --label=com.centurylinklabs.watchtower.enable=false someimage
```

If you need to only include only some containers, pass the --label-enable flag on startup and set the _com.centurylinklabs.watchtower.enable_ label with a value of true for the containers you want to watch.

```docker
LABEL com.centurylinklabs.watchtower.enable="true"
```

Or, it can be specified as part of the `docker run` command line:

```bash
docker run -d --label=com.centurylinklabs.watchtower.enable=true someimage
```

## Remote Hosts

By default, watchtower is set-up to monitor the local Docker daemon (the same daemon running the watchtower container itself). However, it is possible to configure watchtower to monitor a remote Docker endpoint. When starting the watchtower container you can specify a remote Docker endpoint with either the `--host` flag or the `DOCKER_HOST` environment variable:

```bash
docker run -d \
  --name watchtower \
  containrrr/watchtower --host "tcp://10.0.1.2:2375"
```

or

```bash
docker run -d \
  --name watchtower \
  -e DOCKER_HOST="tcp://10.0.1.2:2375" \
  containrrr/watchtower
```

Note in both of the examples above that it is unnecessary to mount the _/var/run/docker.sock_ into the watchtower container.

### Secure Connections

Watchtower is also capable of connecting to Docker endpoints which are protected by SSL/TLS. If you've used _docker-machine_ to provision your remote Docker host, you simply need to volume mount the certificates generated by _docker-machine_ into the watchtower container and optionally specify `--tlsverify` flag.

The _docker-machine_ certificates for a particular host can be located by executing the `docker-machine env` command for the desired host (note the values for the `DOCKER_HOST` and `DOCKER_CERT_PATH` environment variables that are returned from this command). The directory containing the certificates for the remote host needs to be mounted into the watchtower container at _/etc/ssl/docker_.

With the certificates mounted into the watchtower container you need to specify the `--tlsverify` flag to enable verification of the certificate:

```bash
docker run -d \
  --name watchtower \
  -e DOCKER_HOST=$DOCKER_HOST \
  -e DOCKER_CERT_PATH=/etc/ssl/docker \
  -v $DOCKER_CERT_PATH:/etc/ssl/docker \
  containrrr/watchtower --tlsverify
```

## Updating Watchtower

If watchtower is monitoring the same Docker daemon under which the watchtower container itself is running (i.e. if you volume-mounted _/var/run/docker.sock_ into the watchtower container) then it has the ability to update itself. If a new version of the _containrrr/watchtower_ image is pushed to the Docker Hub, your watchtower will pull down the new image and restart itself automatically.

## Notifications

Watchtower can send notifications when containers are updated. Notifications are sent via hooks in the logging system, [logrus](http://github.com/sirupsen/logrus).
The types of notifications to send are passed via the comma-separated option `--notifications` (or corresponding environment variable `WATCHTOWER_NOTIFICATIONS`), which has the following valid values:

- `email` to send notifications via e-mail
- `slack` to send notifications through a Slack webhook
- `msteams` to send notifications via MSTeams webhook

### Settings

- `--notifications-level` (env. `WATCHTOWER_NOTIFICATIONS_LEVEL`): Controls the log level which is used for the notifications. If omitted, the default log level is `info`. Possible values are: `panic`, `fatal`, `error`, `warn`, `info` or `debug`.

### Notifications via E-Mail

To receive notifications by email, the following command-line options, or their corresponding environment variables, can be set:

- `--notification-email-from` (env. `WATCHTOWER_NOTIFICATION_EMAIL_FROM`): The e-mail address from which notifications will be sent.
- `--notification-email-to` (env. `WATCHTOWER_NOTIFICATION_EMAIL_TO`): The e-mail address to which notifications will be sent.
- `--notification-email-server` (env. `WATCHTOWER_NOTIFICATION_EMAIL_SERVER`): The SMTP server to send e-mails through.
- `--notification-email-server-tls-skip-verify` (env. `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_TLS_SKIP_VERIFY`): Do not verify the TLS certificate of the mail server. This should be used only for testing.
- `--notification-email-server-port` (env. `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT`): The port used to connect to the SMTP server to send e-mails through. Defaults to `25`.
- `--notification-email-server-user` (env. `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER`): The username to authenticate with the SMTP server with.
- `--notification-email-server-password` (env. `WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD`): The password to authenticate with the SMTP server with.

Example:

```bash
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e WATCHTOWER_NOTIFICATIONS=email \
  -e WATCHTOWER_NOTIFICATION_EMAIL_FROM=fromaddress@gmail.com \
  -e WATCHTOWER_NOTIFICATION_EMAIL_TO=toaddress@gmail.com \
  -e WATCHTOWER_NOTIFICATION_EMAIL_SERVER=smtp.gmail.com \
  -e WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER=fromaddress@gmail.com \
  -e WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD=app_password \
  containrrr/watchtower
```

### Notifications through Slack webhook

To receive notifications in Slack, add `slack` to the `--notifications` option or the `WATCHTOWER_NOTIFICATIONS` environment variable.

Additionally, you should set the Slack webhook url using the `--notification-slack-hook-url` option or the `WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL` environment variable.

By default, watchtower will send messages under the name `watchtower`, you can customize this string through the `--notification-slack-identifier` option or the `WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER` environment variable.

Other, optional, variables include:

- `--notification-slack-channel` (env. `WATCHTOWER_NOTIFICATION_SLACK_CHANNEL`): A string which overrides the webhook's default channel. Example: #my-custom-channel.
- `--notification-slack-icon-emoji` (env. `WATCHTOWER_NOTIFICATION_SLACK_ICON_EMOJI`): An [emoji code](https://www.webpagefx.com/tools/emoji-cheat-sheet/) string to use in place of the default icon.
- `--notification-slack-icon-url` (env. `WATCHTOWER_NOTIFICATION_SLACK_ICON_URL`): An icon image URL string to use in place of the default icon.

Example:

```bash
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e WATCHTOWER_NOTIFICATIONS=slack \
  -e WATCHTOWER_NOTIFICATION_SLACK_HOOK_URL="https://hooks.slack.com/services/xxx/yyyyyyyyyyyyyyy" \
  -e WATCHTOWER_NOTIFICATION_SLACK_IDENTIFIER=watchtower-server-1 \
  -e WATCHTOWER_NOTIFICATION_SLACK_CHANNEL=#my-custom-channel \
  -e WATCHTOWER_NOTIFICATION_SLACK_ICON_EMOJI=:whale: \
  -e WATCHTOWER_NOTIFICATION_SLACK_ICON_URL=<icon url> \
  containrrr/watchtower
```

### Notifications via MSTeams incoming webhook

To receive notifications in MSTeams channel, add `msteams` to the `--notifications` option or the `WATCHTOWER_NOTIFICATIONS` environment variable.

Additionally, you should set the MSTeams webhook url using the `--notification-msteams-hook` option or the `WATCHTOWER_NOTIFICATION_MSTEAMS_HOOK_URL` environment variable.

MSTeams notifier could send keys/values filled by `log.WithField` or `log.WithFields` as MSTeams message facts. To enable this feature add `--notification-msteams-data` flag or set `WATCHTOWER_NOTIFICATION_MSTEAMS_USE_LOG_DATA=true` environment variable.

Example:

```bash
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e WATCHTOWER_NOTIFICATIONS=msteams \
  -e WATCHTOWER_NOTIFICATION_MSTEAMS_HOOK_URL="https://outlook.office.com/webhook/xxxxxxxx@xxxxxxx/IncomingWebhook/yyyyyyyy/zzzzzzzzzz" \
  -e WATCHTOWER_NOTIFICATION_MSTEAMS_USE_LOG_DATA=true \
  containrrr/watchtower
```

## Contributors

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore -->
<table><tr><td align="center"><a href="http://codelica.com"><img src="https://avatars3.githubusercontent.com/u/386101?v=4" width="100px;" alt="James"/><br /><sub><b>James</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=Codelica" title="Tests">⚠️</a> <a href="#ideas-Codelica" title="Ideas, Planning, & Feedback">🤔</a></td><td align="center"><a href="https://kopfkrieg.org"><img src="https://avatars2.githubusercontent.com/u/5047813?v=4" width="100px;" alt="Florian"/><br /><sub><b>Florian</b></sub></a><br /><a href="#review-KopfKrieg" title="Reviewed Pull Requests">👀</a> <a href="https://github.com/containrrr/watchtower/commits?author=KopfKrieg" title="Documentation">📖</a></td><td align="center"><a href="https://github.com/bdehamer"><img src="https://avatars1.githubusercontent.com/u/398027?v=4" width="100px;" alt="Brian DeHamer"/><br /><sub><b>Brian DeHamer</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=bdehamer" title="Code">💻</a> <a href="#maintenance-bdehamer" title="Maintenance">🚧</a></td><td align="center"><a href="https://github.com/rosscado"><img src="https://avatars1.githubusercontent.com/u/16578183?v=4" width="100px;" alt="Ross Cadogan"/><br /><sub><b>Ross Cadogan</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=rosscado" title="Code">💻</a></td><td align="center"><a href="https://github.com/stffabi"><img src="https://avatars0.githubusercontent.com/u/9464631?v=4" width="100px;" alt="stffabi"/><br /><sub><b>stffabi</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=stffabi" title="Code">💻</a> <a href="#maintenance-stffabi" title="Maintenance">🚧</a></td><td align="center"><a href="https://github.com/ATCUSA"><img src="https://avatars3.githubusercontent.com/u/3581228?v=4" width="100px;" alt="Austin"/><br /><sub><b>Austin</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=ATCUSA" title="Documentation">📖</a></td><td align="center"><a href="https://labs.ctl.io"><img src="https://avatars2.githubusercontent.com/u/6181487?v=4" width="100px;" alt="David Gardner"/><br /><sub><b>David Gardner</b></sub></a><br /><a href="#review-davidgardner11" title="Reviewed Pull Requests">👀</a> <a href="https://github.com/containrrr/watchtower/commits?author=davidgardner11" title="Documentation">📖</a></td></tr><tr><td align="center"><a href="https://github.com/dolanor"><img src="https://avatars3.githubusercontent.com/u/928722?v=4" width="100px;" alt="Tanguy ⧓ Herrmann"/><br /><sub><b>Tanguy ⧓ Herrmann</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=dolanor" title="Code">💻</a></td><td align="center"><a href="https://github.com/rdamazio"><img src="https://avatars3.githubusercontent.com/u/997641?v=4" width="100px;" alt="Rodrigo Damazio Bovendorp"/><br /><sub><b>Rodrigo Damazio Bovendorp</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=rdamazio" title="Code">💻</a> <a href="https://github.com/containrrr/watchtower/commits?author=rdamazio" title="Documentation">📖</a></td><td align="center"><a href="https://www.taisun.io/"><img src="https://avatars3.githubusercontent.com/u/1852688?v=4" width="100px;" alt="Ryan Kuba"/><br /><sub><b>Ryan Kuba</b></sub></a><br /><a href="#infra-thelamer" title="Infrastructure (Hosting, Build-Tools, etc)">🚇</a></td><td align="center"><a href="https://github.com/cnrmck"><img src="https://avatars2.githubusercontent.com/u/22061955?v=4" width="100px;" alt="cnrmck"/><br /><sub><b>cnrmck</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=cnrmck" title="Documentation">📖</a></td><td align="center"><a href="http://harrywalter.co.uk"><img src="https://avatars3.githubusercontent.com/u/338588?v=4" width="100px;" alt="Harry Walter"/><br /><sub><b>Harry Walter</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=haswalt" title="Code">💻</a></td><td align="center"><a href="http://projectsperanza.com"><img src="https://avatars3.githubusercontent.com/u/74515?v=4" width="100px;" alt="Robotex"/><br /><sub><b>Robotex</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=Robotex" title="Documentation">📖</a></td><td align="center"><a href="http://geraldpape.io"><img src="https://avatars0.githubusercontent.com/u/1494211?v=4" width="100px;" alt="Gerald Pape"/><br /><sub><b>Gerald Pape</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=ubergesundheit" title="Documentation">📖</a></td></tr><tr><td align="center"><a href="https://github.com/fomk"><img src="https://avatars0.githubusercontent.com/u/17636183?v=4" width="100px;" alt="fomk"/><br /><sub><b>fomk</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=fomk" title="Code">💻</a></td><td align="center"><a href="https://github.com/svengo"><img src="https://avatars3.githubusercontent.com/u/2502366?v=4" width="100px;" alt="Sven Gottwald"/><br /><sub><b>Sven Gottwald</b></sub></a><br /><a href="#infra-svengo" title="Infrastructure (Hosting, Build-Tools, etc)">🚇</a></td><td align="center"><a href="https://liberapay.com/techknowlogick/"><img src="https://avatars1.githubusercontent.com/u/164197?v=4" width="100px;" alt="techknowlogick"/><br /><sub><b>techknowlogick</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=techknowlogick" title="Code">💻</a></td><td align="center"><a href="http://log.c5t.org/about/"><img src="https://avatars1.githubusercontent.com/u/1449568?v=4" width="100px;" alt="waja"/><br /><sub><b>waja</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=waja" title="Documentation">📖</a></td><td align="center"><a href="http://scottalbertson.com"><img src="https://avatars2.githubusercontent.com/u/154463?v=4" width="100px;" alt="Scott Albertson"/><br /><sub><b>Scott Albertson</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=salbertson" title="Documentation">📖</a></td><td align="center"><a href="https://github.com/huddlesj"><img src="https://avatars1.githubusercontent.com/u/11966535?v=4" width="100px;" alt="Jason Huddleston"/><br /><sub><b>Jason Huddleston</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=huddlesj" title="Documentation">📖</a></td><td align="center"><a href="https://npstr.space/"><img src="https://avatars3.githubusercontent.com/u/6048348?v=4" width="100px;" alt="Napster"/><br /><sub><b>Napster</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=napstr" title="Code">💻</a></td></tr><tr><td align="center"><a href="https://github.com/darknode"><img src="https://avatars1.githubusercontent.com/u/809429?v=4" width="100px;" alt="Maxim"/><br /><sub><b>Maxim</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=darknode" title="Code">💻</a> <a href="https://github.com/containrrr/watchtower/commits?author=darknode" title="Documentation">📖</a></td><td align="center"><a href="https://schmitt.cat"><img src="https://avatars0.githubusercontent.com/u/17984549?v=4" width="100px;" alt="Max Schmitt"/><br /><sub><b>Max Schmitt</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=mxschmitt" title="Documentation">📖</a></td><td align="center"><a href="https://github.com/cron410"><img src="https://avatars1.githubusercontent.com/u/3082899?v=4" width="100px;" alt="cron410"/><br /><sub><b>cron410</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=cron410" title="Documentation">📖</a></td><td align="center"><a href="https://github.com/Cardoso222"><img src="https://avatars3.githubusercontent.com/u/7026517?v=4" width="100px;" alt="Paulo Henrique"/><br /><sub><b>Paulo Henrique</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=Cardoso222" title="Documentation">📖</a></td><td align="center"><a href="https://coded.io"><img src="https://avatars0.githubusercontent.com/u/107097?v=4" width="100px;" alt="Kaleb Elwert"/><br /><sub><b>Kaleb Elwert</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=belak" title="Documentation">📖</a></td><td align="center"><a href="https://github.com/wmbutler"><img src="https://avatars1.githubusercontent.com/u/1254810?v=4" width="100px;" alt="Bill Butler"/><br /><sub><b>Bill Butler</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=wmbutler" title="Documentation">📖</a></td><td align="center"><a href="https://www.mariotacke.io"><img src="https://avatars2.githubusercontent.com/u/4942019?v=4" width="100px;" alt="Mario Tacke"/><br /><sub><b>Mario Tacke</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=mariotacke" title="Code">💻</a></td></tr><tr><td align="center"><a href="https://markwoodbridge.com"><img src="https://avatars2.githubusercontent.com/u/1101318?v=4" width="100px;" alt="Mark Woodbridge"/><br /><sub><b>Mark Woodbridge</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=mrw34" title="Code">💻</a></td><td align="center"><a href="http://www.arcticbit.se"><img src="https://avatars0.githubusercontent.com/u/1596025?v=4" width="100px;" alt="Simon Aronsson"/><br /><sub><b>Simon Aronsson</b></sub></a><br /><a href="https://github.com/containrrr/watchtower/commits?author=simskij" title="Code">💻</a> <a href="#maintenance-simskij" title="Maintenance">🚧</a> <a href="#review-simskij" title="Reviewed Pull Requests">👀</a></td></tr></table>

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind welcome!
