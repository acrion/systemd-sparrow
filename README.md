# systemd-sparrow
Systemd with sparrow concept


# Code example



`/etc/systemd/system/multi-user.target.wants/beans.service`

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
