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
##################### INTERFACE
ENV=${ENV:-development}

##################### usage
# FYI: all cmds create .env.$ENV.compose.[json,yaml]
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
##################### INTERFACE
ENV=${ENV:-development}

##################### usage
# FYI: all cmds create .env.$ENV.compose.[json,yaml]
# containing the canonical compose config


# stop, rm images and volumes, then recreate compose.yml in current dir
> script.reset.compose.sh

# stop, rm images and volumes, then recreate SERVICE_NAME
> script.refresh.compose.sh SERVICE_NAME

```

### script.exec.cunt.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker/script.exec.cunt.sh)

```sh
##################### INTERFACE
# if you prefix all your container names
export CUNT_NAME_PREFIX=nirvai_


##################### usage
# exec bash, else sh into $CONTAINER_NAME or $SERVICE_PREFIX$CONTAINER_NAME
> script.exec.cunt.sh CONTAINER_NAME

```

### script.log.cunt.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/docker/script.log.cunt.sh)

```sh
##################### INTERFACE
# if you prefix all your container names
export CUNT_NAME_PREFIX=nirvai_


##################### usage
# get the logs on disk for a container, useful if docker logs returns null
> script.log.cunt.sh disk CONTAINER_NAME

```

### script.registry.sh

- actively used for working with a locally docker registry
- primarly for automating the creation of images accessible by nomad

```sh
###################### setup your /etc/hosts
# e.g. to use a registry at dev.nirv.ai:5000
# add the following to /etc/hosts
127.0.0.1 dev.nirv.ai
# checkout ../letsencrypt/README.md for configuring a TLS cert pointing to dev.nirv.ai
######################


###################### interface
# TODO: explain concat: $REG_CERTS_PATH/$REG_SUBD.$REG_DOMAIN/*.pem
export REG_CERTS_PATH=/etc/ssl/certs
export REG_DOMAIN=nirv.ai
export REG_SUBD=dev
export REG_HOST_PORT=5000
## your registry will be available at dev.nirv.ai:5000
## will auto tag images to that registry
## will auto remove images sourced from hub to push to that registry

###################### basic workflow
# start the registry
script.registry.sh run
# push running containers to registry
script.registry.sh tag_running
# or tag a specific image
script.registry.sh tag someImageName:development

##################### usage
## ensure the ENV vars are set
## its setup to point for local development at dev.nirv.ai
script.registry.sh CMD

> run: runs a registry
> reset: purges and restarts a registry
> tag: tag a single image and pushes it to registry
> tag_running: tags the image of all running containers and pushes all to registry

```
