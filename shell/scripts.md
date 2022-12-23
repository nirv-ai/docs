# NIRVAI SCRIPTS

- useful scripts for working within a monorepo, e.g. [turborepo](https://turbo.build/repo/docs)
- additional readme coming soon
- [source code](https://github.com/nirv-ai/scripts)

## TLDR

- requires the following to be available
  - [JQ](https://stedolan.github.io/jq/)
  - [YQ](https://github.com/mikefarah/yq)
  - bash, or zsh/fish/etc set to `export bash=zsh`
    - moving to POSIX shell soon
  - each script requires the service it works with, e.g. registry.sh requires docker, vault requires vault, etc
- clone the repo and symlink the scripts to the root of your repo
  - some scripts are useful in the root of a specific app, e.g. nmd.sh should be wherever you execute nomad

## script.registry.sh

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

## script.nmd.sh

- actively used for working with nomad
- requires you have a local docker registry setup, see registry above
- access the UI @ [https://mad.nirv.ai:4646](https://mad.nirv.ai:4646/ui/jobs)
- requires you've setup [TLS with cloudflare cfssl](https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls)

```sh
###################### helpful links
# @see https://github.com/hashicorp/nomad
# @see https://discuss.hashicorp.com/t/failed-to-find-plugin-bridge-in-path/3095
## ^ need to enable cni plugin
# @see https://developer.hashicorp.com/nomad/docs/drivers/docker#enabled-1
## ^ need to enable bind mounts
# @see https://developer.hashicorp.com/nomad/docs/drivers/docker#allow_caps
## ^ for vault you need to enable cap_add ipc_lock
## ^ for debugging set it to "all"
# @see registry.sh
# failed to find docker auth
## update your server config to not fault on auth errors, or set an auth (see nomad && docker docs)
######################

###################### interface:
# export as ENV vars to enable cli use without going through script.nmd.sh
NOMAD_ADDR_SUBD=${ENV:-dev}
NOMAD_ADDR_HOST=${NOMAD_ADDR_HOST:-nirv.ai}
NOMAD_SERVER_PORT="${NOMAD_SERVER_PORT:-4646}"
NOMAD_ADDR="${NOMAD_ADDR:-https://${NOMAD_ADDR_SUBD}.${NOMAD_ADDR_HOST}:${NOMAD_SERVER_PORT}}"
NOMAD_CACERT="${NOMAD_CACERT:-./tls/nomad-ca.pem}"
NOMAD_CLIENT_CERT="${NOMAD_CLIENT_CERT:-./tls/cli.pem}"
NOMAD_CLIENT_KEY="${NOMAD_CLIENT_KEY:-./tls/cli-key.pem}"

###################### basic workflow
########### cd nirvai/core
# refresh containers and upsert .env.${ENV}.compose.{json,yaml}
./script.refresh.compose.sh


# ensure you've completed steps in ./script.registry.sh (see above)
# start the registry
./script.registry.sh run
# tag-and-push all images backing running containers to the registry
./script.registry.sh tag_running
# stop all running containers (nomad uses the images in the registry)
docker compose down


########### cd ./apps/nirvai-core-nomad/dev
# you only need to do this the first time
# symlink the json & yaml files
ln -s ../../../.env.development.compose.* .
# symlink the nomad script (expected to be a sibling of nirvai/core)
ln -s ../../../../scripts/script.nmd.sh .

###################### now you can operate nomad
# start server agent in bg
./script.nmd.sh start s -config=development.server.nomad
# start client agent in bg
./script.nmd.sh start c -config=development.client.nomad
# check the team status
./script.nmd.sh get status team


# if ./development.dev_core.nomad doesnt exist
# create it and get the index number for stdout
./script.nmd.sh get plan dev_core

# deploy job dev_core
./script.nmd.sh run dev_core indexNumber
./script.nmd.sh dockerlogs # [optional] see logs of all running containers

# on error run
./script.nmd.sh get status job jobName # includes all allocationIds
./script.nmd.sh get status loc allocationId # in event of deployment failure
./script.nmd.sh get status node # see client nodes and there ids
./script.nmd.sh dockerlogs # following docker logs of all running containers
nomad alloc exec -i -t -task sidekiq fa2b2ed6 /bin/bash # todo,
nomad alloc exec -i -t -task puma fa2b2ed6 /bin/bash -c "bundle exec rails c" #todo
nomad job history -p job_name # todo

# cleanup
# rm the job
./script.nmd.sh rm dev_core
# kill the team @see https://github.com/noahehall/theBookOfNoah/blob/master/linux/bash_cli_fns/000util.sh
kill_service_by_name nomad
# reset nomad to a green state if you dont plan on using it later
nomad system gc

###################### USAGE
## prefix all cmds with ./script.nmd.sh poop poop poop
## poop being one of the below

# check on the server
get status team
get status all
dockerlogs # following logs of all running containers

# creating stuff
create gossipkey
create job myJobName
get plan myJobName # provides indexNumber

# running stuff
run myJobName indexNumber

# restarting stuff
restart loc allocationId taskName # todo: https://developer.hashicorp.com/nomad/docs/commands/alloc/restart

# execing stuff
exec loc  allocationId cmd .... @ todo https://developer.hashicorp.com/nomad/docs/commands/alloc/exec

# checking on running/failing stuff
get status node # see client nodes and there ids
get status node nodeId # provding a clients nodeId is super helpful; also provides allocationId
get status loc allocationId # super helpful for checking on failed jobs, provides deployment id
get status dep deploymentId # super helpful
get logs jobName deploymentId

# stopping stuff
stop job myJobName
rm myJobName # this purges the job
gc # purge system

```

## Vault Scripts

- [docs are in the docs repo](https://github.com/nirv-ai/docs/tree/main/vault)
