# NIRVai HAProxy

documentation for the HAProxy @ NIRVai

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## setup

- [stats page @ ENV.nirv.ai:8404](https://dev.nirv.ai:8404)

## script.haproxy.sh

### INTERFACE

```sh
# the prefix to all your container names
## we add this to each cmd automatically where apropriate
SERVICE_PREFIX=nirvai

# where all your .cfg files are kept
## e.g. /usr/local/etc/haproxy/configs
HAPROXY_CONFIG_DIR=./configs

```

### basic workflow

```sh
# prefix all cmds with script.haproxy.sh

## validate configuration for running container: nirvai_core_proxy
conf validate core_proxy

## reload configuration for running container: nirvai_core_proxy
conf reload core_proxy

## get server status for running container: nirvai_core_proxy
conf info core_proxy
```
