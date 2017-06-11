# The s6overlayscripts is the init scripts extracted from [s6-overlay](https://github.com/just-containers/s6-overlay).

## Quickstart
  `s6` and `s6-portable-utils` packages should be installed before using this scripts. Run `prep` after copy the scripts into docker image.

## Goals
The project has the following goals:

* Make it easy for image authors to take advantage of s6
* Still operate like other Docker images

## Features

* A simple init process which allows to the end-user execute tasks like initialization (`cont-init.d`), finalization (`cont-finish.d`) as well as fixing ownership permissions (`fix-attrs.d`).
* The s6-overlay provides proper `PID 1` functionality
  * You'll never have zombie processes hanging around in your container, they will be properly cleaned up.
* Multiple processes in a single container
* Able to operate in "The Docker Way"
* Usable with all base images - Ubuntu, CentOS, Fedora, and even Busybox.
* Distributed as a single .tar.gz file, to keep your image's number of layers small.
* Log rotating out-of-the-box through `logutil-service` which uses [`s6-log`](http://skarnet.org/software/s6/s6-log.html) under the hood.

## Our `s6-overlay` based images

We've developed two docker images which can be used as base images:
* [aps6base](https://hub.docker.com/r/hwdm/aps6base/): Based on Alpine latest, it was intended to serve as a general purpose image.

## Init stages

Our overlay init is a properly customized one to run appropriately in containerized environments. This section briefly explains how our stages work but if you want to know how a complete init system should work, please read this article: [How to run s6-svscan as process 1](http://skarnet.org/software/s6/s6-svscan-1.html) by Laurent Bercot.

1. **stage 1**: Its purpose is to prepare the image to enter into the second stage. Among other things, it is responsible for preparing the container environment variables, block the startup of the second stage until `s6` is effectively started, ...
2. **stage 2**: This is where most of the end-user provided files are mean to be executed:
  1. Fix ownership and permissions using `/etc/fix-attrs.d`.
  2. Execute initialization scripts contained in `/etc/cont-init.d`.
  3. Copy user services (`/etc/services.d`) to the folder where s6 is running its supervision and signal it so that it can properly start supervising them.
3. **stage 3**: This is the shutdown stage. Its purpose is to clean everything up, stop services and execute finalization scripts contained in `/etc/cont-finish.d`. This is when our init system stops all container processes, first gracefully using `SIGTERM` and then (after `S6_KILL_GRACETIME`) forcibly using `SIGKILL`. And, of course, it reaps all zombies :-).

## Usage

### Customizing `s6` behaviour

It is possible somehow to tweak `s6` behaviour by providing an already predefined set of environment variables to the execution context:

* `S6_KEEP_ENV` (default = 0): if set, then environment is not reset and whole supervision tree sees original set of env vars. It switches `with-contenv` into noop.
* `S6_LOGGING` (default = 0):
  * **`0`**: Outputs everything to stdout/stderr.
  * **`1`**: Uses an internal `catch-all` logger and persists everything on it, it is located in `/var/log/s6-uncaught-logs`. Nothing would be written to stdout/stderr.
* `S6_BEHAVIOUR_IF_STAGE2_FAILS` (default = 0):
  * **`0`**: Continue silently even if any script (`fix-attrs` or `cont-init`) has failed.
  * **`1`**: Continue but warn with an annoying error message.
  * **`2`**: Stop by sending a termination signal to the supervision tree.
* `S6_KILL_FINISH_MAXTIME` (default = 5000): The maximum time (in milliseconds) a script in `/etc/cont-finish.d` could take before sending a `KILL` signal to it. Take into account that this parameter will be used per each script execution, it's not a max time for the whole set of scripts.
* `S6_KILL_GRACETIME` (default = 3000): How long (in milliseconds) `s6` should wait to reap zombies before sending a `KILL` signal.
* `S6_LOGGING_SCRIPT` (default = "n20 s1000000 T"): This env decides what to log and how, by default every line will prepend with ISO8601, rotated when the current logging file reaches 1mb and archived, at most, with 20 files.
* `S6_CMD_ARG0` (default = not set): Value of this env var will be prepended to any `CMD` args passed by docker. Use it if you are migrting an existing image to a s6-overlay and want to make it a drop-in replacement, then setting this variable to a value of previously used ENTRYPOINT will improve compatibility with the way image is used.
* `S6_FIX_ATTRS_HIDDEN` (default = 0): Controls how `fix-attrs.d` scripts process files and directories.
  * **`0`**: Hidden files and directories are excluded.
  * **`1`**: All files and directories are processed.
* `S6_CMD_WAIT_FOR_SERVICES` (default = 0): In order to proceed executing CMD overlay will wait until services are up. Be aware that up doesn't mean ready. Depending if `notification-fd` was found inside the servicedir overlay will use `s6-svwait -U` or `s6-svwait -u` as the waiting statement.
* `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` (default = 5000): The maximum time (in milliseconds) the services could take to bring up before proceding to CMD executing.
* `S6_READ_ONLY_ROOT` (default = 0): When running in a container whose root filesystem is read-only, set this env to **1** to inform init stage 2 that it should copy user-provided initialization scripts from `/etc` to `/var/run/s6/etc` before it attempts to change permissions, etc. See [Read-Only Root Filesystem](#read-only-root-filesystem) for more information.

### Using `CMD`

Using `CMD` is a really convenient way to take advantage of the s6-overlay. Your `CMD` can be given at build-time in the Dockerfile, or at runtime on the command line, either way is fine - it will be run under the s6 supervisor, and when it fails or exits, the container will exit. You can even run interactive programs under the s6 supervisor!

docker-host $ docker run -ti s6demo /bin/sh
[fix-attrs.d] applying owners & permissions fixes...
[fix-attrs.d] 00-runscripts: applying...
[fix-attrs.d] 00-runscripts: exited 0.
[fix-attrs.d] done.
[cont-init.d] executing container initialization scripts...
[cont-init.d] done.
[services.d] starting services
[services.d] done.
/ # ps
PID   USER     COMMAND
    1 root     s6-svscan -t0 /var/run/s6/services
   21 root     foreground  if   /etc/s6/init/init-stage2-redirfd   foreground    if     s6-echo     [fix-attrs.d] applying owners & permissions fixes.
   22 root     s6-supervise s6-fdholderd
   23 root     s6-supervise s6-svscan-log
   24 nobody   s6-log -bp -- t /var/log/s6-uncaught-logs
   28 root     foreground  s6-setsid  -gq  --  with-contenv  /bin/sh  import -u ? if  s6-echo  --  /bin/sh exited ${?}  foreground  s6-svscanctl  -t
   73 root     /bin/sh
   76 root     ps
/ # exit
/bin/sh exited 0
docker-host $
```

### Fixing ownership & permissions

Sometimes it's interesting to fix ownership & permissions before proceeding because, for example, you have mounted/mapped a host folder inside your container. Our overlay provides a way to tackle this issue using files in `/etc/fix-attrs.d`. This is the pattern format followed by fix-attrs files:

```
Format:
        path recurse account fmode [fmode fallback] dmode
```
* `path`: File or dir path.
* `recurse`: (Set to `true` or `false`) If a folder is found, recurse through all containing files & folders in it.
* `account`: Target account. It's possible to default to fallback `uid:gid` if the account isn't found. For example, `nobody,32768:32768` would try to use the `nobody` account first, then fallback to `uid 32768` instead.
If, for instance, `daemon` account is `UID=2` and `GID=2`, these are the possible values for `account` field:
  * `daemon:                UID=2     GID=2`
  * `daemon,3:4:            UID=2     GID=2`
  * `2:2,3:4:               UID=2     GID=2`
  * `daemon:11111,3:4:      UID=11111 GID=2`
  * `11111:daemon,3:4:      UID=2     GID=11111`
  * `daemon:daemon,3:4:     UID=2     GID=2`
  * `daemon:unexisting,3:4: UID=2     GID=4`
  * `unexisting:daemon,3:4: UID=3     GID=2`
  * `11111:11111,3:4:       UID=11111 GID=11111`
* `fmode`: Target file mode. For example, `0644`.
* `dmode`: Target dir/folder mode. For example, `0755`.

Here you have some working examples:

`/etc/fix-attrs.d/01-mysql-data-dir`:
```
/var/lib/mysql true mysql 0600 0700
```
`/etc/fix-attrs.d/02-mysql-log-dirs`:
```
/var/log/mysql-error-logs true nobody,32768:32768 0644 2700
/var/log/mysql-general-logs true nobody,32768:32768 0644 2700
/var/log/mysql-slow-query-logs true nobody,32768:32768 0644 2700
```

### Executing initialization And/Or finalization tasks

After fixing attributes (through `/etc/fix-attrs.d/`) and just before starting user provided services up (through `/etc/services.d`) our overlay will execute all the scripts found in `/etc/cont-init.d`, for example:

[`/etc/cont-init.d/02-confd-onetime`](https://github.com/just-containers/nginx-loadbalancer/blob/master/rootfs/etc/cont-init.d/02-confd-onetime):
```
#!/usr/bin/execlineb -P

with-contenv
s6-envuidgid nginx
multisubstitute
{
  import -u -D0 UID
  import -u -D0 GID
  import -u CONFD_PREFIX
  define CONFD_CHECK_CMD "/usr/sbin/nginx -t -c {{ .src }}"
}
confd --onetime --prefix="${CONFD_PREFIX}" --tmpl-uid="${UID}" --tmpl-gid="${GID}" --tmpl-src="/etc/nginx/nginx.conf.tmpl" --tmpl-dest="/etc/nginx/nginx.conf" --tmpl-check-cmd="${CONFD_CHECK_CMD}" etcd
```

### Writing a service script

Creating a supervised service cannot be easier, just create a service directory with the name of your service into `/etc/services.d` and put a `run` file into it, this is the file in which you'll put your long-lived process execution. You're done! If you want to know more about s6 supervision of servicedirs take a look to [`servicedir`](http://skarnet.org/software/s6/servicedir.html) documentation. A simple example would look like this:

`/etc/services.d/myapp/run`:
```
#!/usr/bin/execlineb -P
nginx -g "daemon off;"
```

### Writing an optional finish script

By default, services created in `/etc/services.d` will automatically restart. If a service should bring the container down, you'll need to write a `finish` script that does that. Here's an example finish script:

`/etc/services.d/myapp/finish`:
```
#!/usr/bin/execlineb -S0

s6-svscanctl -t /var/run/s6/services
```

It's possible to do more advanced operations - for example, here's a script from @smebberson that only brings down the service when it crashes:

`/etc/services.d/myapp/finish`:
```
#!/usr/bin/execlineb -S1
if { s6-test ${1} -ne 0 }
if { s6-test ${1} -ne 256 }

s6-svscanctl -t /var/run/s6/services
```

### Logging

Our overlay provides a way to handle logging easily since `s6` already provides logging mechanisms out-of-the-box via [`s6-log`](http://skarnet.org/software/s6/s6-log.html)!. We also provide a helper utility called `logutil-service` to make logging a matter of calling one binary. This helper does the following things:
- read how s6-log should proceed reading the logging script contained in `S6_LOGGING_SCRIPT`
- drop privileges to the `nobody` user (defaulting to `32768:32768` if it doesn't exist)
- clean all the environments variables
- initiate logging by executing s6-log :-)

This example will send all the log lines present in stdin (following the rules described in `S6_LOGGING_SCRIPT`) to `/var/log/myapp`:

`/etc/services.d/myapp/log/run`:
```
#!/bin/sh
exec logutil-service /var/log/myapp
```

If, for instance, you want to use a fifo instead of stdin as an input, write your log services as follows:

`/etc/services.d/myapp/log/run`:
```
#!/bin/sh
exec logutil-service -f /var/run/myfifo /var/log/myapp
```

### Dropping privileges

When it comes to executing a service, no matter it's a service or a logging service, a very good practice is to drop privileges before executing it. `s6` already includes utilities to do exactly these kind of things:

In `execline`:

```
#!/usr/bin/execlineb -P
s6-setuidgid daemon
myservice
```

In `sh`:

```
#!/bin/sh
exec s6-setuidgid daemon myservice
```

If you want to know more about these utilities, please take a look to: [`s6-setuidgid`](http://skarnet.org/software/s6/s6-setuidgid.html), [`s6-envuidgid`](http://skarnet.org/software/s6/s6-envuidgid.html) and [`s6-applyuidgid`](http://skarnet.org/software/s6/s6-applyuidgid.html).

### Container environment

If you want your custom script to have container environments available just make use of `with-contenv` helper, which will push all of those into your execution environment, for example:

`/etc/cont-init.d/01-contenv-example`:
```
#!/usr/bin/with-contenv sh
echo $MYENV
```

This script will output whatever the `MYENV` enviroment variable contains.

### Read-Only Root Filesystem

Recent versions of Docker allow running containers with a read-only root filesystem. During init stage 2, the overlay modifies permissions for user-provided files in `cont-init.d`, etc. If the root filesystem is read-only, you can set `S6_READ_ONLY_ROOT=1` to inform stage 2 that it should first copy user-provided files to its work area in `/var/run/s6` before attempting to change permissions.

This of course assumes that at least `/var` is backed by a writeable filesystem with execute privileges. This could be done with a `tmpfs` filesystem as follows:

```
docker run -e S6_READ_ONLY_ROOT=1 --read-only --tmpfs /var:rw,exec [image name]
```

**NOTE**: When using `S6_READ_ONLY_ROOT=1` you should _avoid using symbolic links_ in `fix-attrs.d`, `cont-init.d`, `cont-finish.d`, and `services.d`. Due to limitations of `s6`, symbolic links will be followed when these directories are copied to `/var/run/s6`, resulting in unexpected duplication.

## Known issues and workarounds

### `/bin` and `/sbin` are symlinks

Some Linux distros (like CentOS 7) have started replacing `/bin` with a symlink
to `/usr/bin` (and the same for `/sbin -> /usr/sbin`). When you extract the
tarball, these symlinks are overwritten, so important programs like `/bin/sh`
disappear.

The current workaround is to extract the tarball in two steps:

```
RUN tar xzf /tmp/s6-overlay-amd64.tar.gz -C / --exclude="./bin" --exclude="./sbin" && \
    tar xzf /tmp/s6-overlay-amd64.tar.gz -C /usr ./bin ./sbin
```

This will prevent tar from deleting those `/bin` and `/sbin` symlinks, and
everything will work as normal.


## only support `root` users

* For now, `s6-overlay` doesn't support running it with a user different than `root`, so consequently Dockerfile `USER` directive is not supported (except `USER root` of course ;P).

### Authors & License

See https://github.com/just-containers/s6-overlay
