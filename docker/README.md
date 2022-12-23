# nirvai docker

documentation for the NIRVai platform

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## scripts

- [scripting architecture & guidance](../scripts/README.md)

### script.registry.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.registry.sh)
- actively used for working with a private docker registry on localhost

```sh
###################### READ FIRST
# @see [repo] https://github.com/distribution/distribution
# @see [docs] https://github.com/docker/docs/tree/main/registry
# @see https://www.marcusturewicz.com/blog/build-and-publish-docker-images-with-github-packages/
## @see https://docs.github.com/en/actions/publishing-packages/publishing-docker-images
######################

###################### FYI
# setup for a local registry for development
# but definitely recommend canceling disney plus (but keep netflix, just sayin)
# so you can afford $5 (...$7) private registry with docker hub
## from hub: You are expected to be familiar with systems
## availability and scalability, logging and log processing,
## systems monitoring, and security 101. Strong understanding
## of http and overall network communications,
## plus familiarity with golang are certainly useful
## as well for advanced operations or hacking.
######################

###################### setup your /etc/hosts
# e.g. to use a registry at dev.nirv.ai:5000
# add the following to /etc/hosts
127.0.0.1 dev.nirv.ai
# checkout /letencrypt dir for configuring a TLS cert pointing to dev.nirv.ai
######################


###################### interface
export REG_CERTS_PATH=apps/nirvai-core-letsencrypt/dev-nirv-ai
export REG_DOMAIN=nirv.ai
export REG_SUBD=dev
export REG_HOST_PORT=5000
## your registry will be available at dev.nirv.ai:5000
## will auto tag images to that registry
## will auto remove images sourced from hub to push to that registry

###################### basic workflow
# start the registry
./script.registry.sh run
# push running containers to registry
./script.registry.sh tag_running
# or tag a specific image
./script.registry.sh tag someImageName:development

##################### usage
## ensure the ENV vars are set
## its setup to point for local development at dev.nirv.ai
./script.registry.sh poop

> run: runs a registry
> reset: purges and restarts a registry
> tag: tag a single image and pushes it to registry
> tag_running: tags the image of all running containers and pushes all to registry

```

### script.compose.refresh.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.compose.refresh.sh)

### script.compose.reset.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.compose.reset.sh)

### script.exec.container.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.exec.sh)
