**icinga2-check-submitter** is a simple Python script used to invoke passive
checks and submit their results to an Icinga2 server using the Icinga2 API. It
is meant to be invoked via cron or a systemd timer.

## Installation

You need to create an API user on the Icinga2 server, and install the script
and a configuration file on the client you intend to monitor.

### Icinga2 Server

Add an Icinga2 API user in `/etc/icinga2/conf.d/api-users.conf`:

```
object ApiUser "changeme" {
    password = "some strong randomly generated password"
    permissions = [ "actions/process-check-result" ]
}
```

You can create an individual user for each machine running passive checks or
use a shared user for all. You may wish to add a
[filter](https://icinga.com/docs/icinga2/latest/doc/12-icinga2-api/#icinga2-api-permissions)
so that an ApiUser can only submit check results for a subset of hosts (or even
just a single host).

### Client

Create a JSON config in `/etc/nagios/icinga2-passive-checks.json`. It must
contain five entries:

* `host` is the hostname for which passive checks are submitted.
* `passive_ping` determines whether the passive check submission
  includes a `hostalive` result. Use this for machines which are not reachable
  from the Icinga2 server (i.e., which are not covered by Icinga2's hostalive
  ping check).
* `checks` is a dict of checks, e.g. `"disk /": "/usr/lib/nagios/plugins/check_disk -w 8% -c 2% -p /"`.
* `api` is the Icinga2 server's API endpoint.
* `auth` is a list containing API username and password (see above).

Example:

```
{
  "host": "spacetime.derf0.net",
  "passive_ping": false,
  "checks" : {
    "disk /": "/usr/lib/nagios/plugins/check_disk -w 8% -c 2% -p /",
    "disk /home": "/usr/lib/nagios/plugins/check_disk -w 8% -c 2% -p /",
    "users": "/usr/lib/nagios/plugins/check_users -w 20 -c 30",
    "load": "/usr/lib/nagios/plugins/check_load --warning=20,15,10 --critical=30,25,20",
    "procs": "/usr/lib/nagios/plugins/check_procs -w 400 -c 500"
  },
  "api": "https://icinga2.example.org:5665/v1/actions/process-check-result",
  "auth": ["changeme", "some strong randomly generated password"]
}
```

Additionally, install `icinga2-run-checks` somewhere and add a crontab entry or
systemd timer to run it periodically. For example, in
`/etc/cron.d/icinga2-run-checks`:

```
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

*/5 * * * * nagios /usr/local/lib/icinga2-run-checks cron
```

You may need to add a nagios user first, e.g. (on Debian) `adduser --quiet
--system --no-create-home --disabled-login --shell /bin/sh --home
/var/lib/nagios nagios`.
