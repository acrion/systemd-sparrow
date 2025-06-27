# systemd-sparrow
Systemd with sparrow concept


# Code example

The content of file `/etc/systemd/system/multi-user.target.wants/beans.service`:

```raku
#!raku

task-run "my service", "systemd", %(
    after-service => [
        "nginx",
        "mysql",
    ],
    after-target => [
        "syslog",
        "network"
    ],
    env => %(
        :RACK_ENV<production>,
    ),
    description => "Cool Beans",
    start => "/usr/local/bin/beans start -f",
    stop => "/usr/local/bin/beans stop",
    reload => "kill -s HUP /var/run/beans.pid",
);
```

# Flow

- Systemd instead (*) of parsing /etc/systemd/system/multi-user.target.wants/beans.service as a regular systemd configuration file, spawns a Raku oneliner: `raku -MSparrow6::DSL -e /etc/systemd/system/multi-user.target.wants/beans.service`

- As a result of the Raku onliner execution `/tmp/beans.service.out` file has been created with **native** systemd configuration data

- Systemd parses it as a normal systemd unit file and continues its work


\* the dicision whether to parse /etc/systemd/system/multi-user.target.wants/beans.service as a regular systemd unit file or spawns intermediate Raku online could be taken by analyzing the first line of could, so there is a `#!raku` shebang this is a Raku/Sparrow file, not native unit file.

# Explanation

## Systemd sparrow plugin

Sparrow provides a systemd sparrow plugin which gets run as function inside Rakud code - `task-run "bla bla bla", "systemd", %PARAMS`, this plugin generates desired systemd unit code in natived systemd format suitable for systemd parsing. This interaction is hidden from end user who writes unitd code in oure Raku/Sparrow DSL


# Advantages

## Flexibility 

Because it's pure Raku, this sky is a limit, so for example:

```raku
#!raku

task-run "my service", "systemd", %(
    after-service => [
        "nginx",
        "mysql",
    ],
    after-target => [
        "syslog",
        "network"
    ],
    env => %(
        MemoryMax => ($*KERNEL.memory * 0.25).Int,
    ),
    description => "Cool Beans",
    start => "/usr/local/bin/beans start -f",
    stop => "/usr/local/bin/beans stop",
    reload => "kill -s HUP /var/run/beans.pid",
);
```

## Reusability

Because Sparrow comes with a concept of [plugins](https://sparrowhub.io) - small reusable tasks written on Raku/Bash/Python/Perl

We could think of creation of "precooked" systemd units for some well known services:


### Nginx

`/etc/systemd/system/multi-user.target.wants/nginx.service`:

```raku
#!raku
task-run "nginx service", "systemd-nginx";
```

`/etc/systemd/system/multi-user.target.wants/mariadb.service`:

```raku
#!raku
task-run "mariadb service", "systemd-mariadb";
```

Etc ...
