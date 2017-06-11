# docker image: alpinebase [Layer 1]

## Description
Images        --->  git directory
hwdm/apbase   --->  alpinebase: enhanced alpine base image
hwdm/apglibc  --->  alpineglibc: alpinebase + glibc
hwdm/aprunit  --->  aprunit: alpinebase + runit + multiple procs manangement scripts
hwdm/aps6base --->  aps6base: alpinebase + s6 + multiple procs manangement scripts

## Built Images:
See https://hub.docker.com/u/hwdm/

## Layout

Shared Users and Groups: `www-data:www-data`

Possible volumes and their purposes:
`/run/<servname>[Owner:www-data:www-data,Temporary]` is for container-specific performance related pid, cache or tmp files.
`/etc/<servname>` is for inter-container-shared service configuration files etc.
`/var/run/<servname>[Owner:www-data:www-data]` is for inter-container-shared sock files etc.
`/var/log/<servname>[Owner:www-data:www-data]` is for inter-container-shared log files etc.
`/var/www/<sitename>[Owner:www-data:www-data]` is for inter-container-shared web application source code and configurations.
`/var/opt/<appname>` is for inter-container-shared web application backend source code and configurations.

# License
[Apache 2.0](https://www.tldrlegal.com/l/apache2)
