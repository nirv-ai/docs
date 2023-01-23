# nirvai docker

documentation for the NIRVai platform

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## available scripts

- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker)

### script.refresh.compose.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker/script.refresh.compose.sh)

```sh
##################### usage
# FYI: all cmds create .env.compose.[json,yaml]
# containing the canonical compose config


# stop, rm, then recreate compose.yml in current dir
> script.refresh.compose.sh

# stop, rm, then recreate SERVICE_NAME
> script.refresh.compose.sh SERVICE_NAME

# stop, rm, build image, and recreate CONTAINER_NAME
> script.refresh.compose.sh SERVICE_NAME 1
```

### script.reset.compose.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker/script.reset.compose.sh)

```sh
##################### usage
# FYI: all cmds create .env.compose.[json,yaml]
# containing the canonical compose config


# stop, rm images and volumes, then recreate compose.yml in current dir
> script.reset.compose.sh

# stop, rm images and volumes, then recreate SERVICE_NAME
> script.reset.compose.sh SERVICE_NAME

```

### script.exec.cunt.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker/script.exec.cunt.sh)

```sh
##################### INTERFACE
# set to COMPOSE_PROJECT_NAME (or to empty string)
# be sure to add the trailing `-` after your project name
export CUNT_PREFIX=nirvai-


##################### usage
## exec into cunt using bash, else sh:

# using the exact container name
> script.exec.cunt.sh backend-api-us-east-1a-1

# using the service name: execs the first matching container
> script.exec.cunt.sh backend-api
> script.exec.cunt.sh b # the first container whose name includes b

```

### script.log.cunt.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker/script.log.cunt.sh)

```sh
##################### INTERFACE
# set to COMPOSE_PROJECT_NAME (or to empty string)
# be sure to add the trailing `-` after your project name
export CUNT_PREFIX=nirvai-


##################### usage
# get the logs on disk for a container, useful if docker logs returns null
> script.log.cunt.sh disk CONTAINER_NAME

```

### script.registry.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker/script.registry.sh)

```sh
###################### setup your /etc/hosts
# e.g. to use a registry at dev.nirv.ai:5000
# add the following to /etc/hosts
127.0.0.1 dev.nirv.ai
######################


###################### interface
export CERTS_DIR_CUNT=/run/secrets
export CERTS_DIR_HOST=/etc/ssl/certs
export REG_HOST_PORT=5000
export REG_HOSTNAME=dev.nirv.ai # echo 127.0.0.1 dev.nirv.ai >> /etc/hosts
export REG_NAME=registry
export REG_PRVKEY_NAME=privkey.pem
export REG_PUBKEY_NAME=fullchain.pem
export REG_VOL_NAME=registry_volume

###################### basic workflow
# start the registry
script.registry.sh run
# push running containers to registry
script.registry.sh tag-running
# or tag a specific image
script.registry.sh tag someImageName:development

##################### usage
## ensure the ENV vars are set
## its setup to point for local development at dev.nirv.ai
script.registry.sh CMD

> run: runs a registry
> reset: purges and restarts a registry
> tag: tag a single image and pushes it to registry
> tag-running: tags the image of all running containers and pushes all to registry

```
