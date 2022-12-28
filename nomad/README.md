# nirvai docs

documentation for the NIRVai platform

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## scripts

- [scripting architecture & guidance](../scripts/README.md)

### script.nmd.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/nomad)
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
refresh.compose.sh


# ensure you've completed steps in registry.sh (see above)
# start the registry
registry.sh run
# tag-and-push all images backing running containers to the registry
registry.sh tag_running
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
nmd.sh start s -config=development.server.nomad
# start client agent in bg
nmd.sh start c -config=development.client.nomad
# check the team status
nmd.sh get status team


# if ./development.dev_core.nomad doesnt exist
# create it and get the index number for stdout
nmd.sh get plan dev_core

# deploy job dev_core
nmd.sh run dev_core indexNumber
nmd.sh dockerlogs # [optional] see logs of all running containers

# on error run
nmd.sh get status job jobName # includes all allocationIds
nmd.sh get status loc allocationId # in event of deployment failure
nmd.sh get status node # see client nodes and there ids
nmd.sh dockerlogs # following docker logs of all running containers
nomad alloc exec -i -t -task sidekiq fa2b2ed6 /bin/bash # todo,
nomad alloc exec -i -t -task puma fa2b2ed6 /bin/bash -c "bundle exec rails c" #todo
nomad job history -p job_name # todo

# cleanup
# rm the job
nmd.sh rm dev_core
# kill the team @see https://github.com/noahehall/theBookOfNoah/blob/master/linux/bash_cli_fns/000util.sh
kill_service_by_name nomad
# reset nomad to a green state if you dont plan on using it later
nomad system gc

###################### USAGE
## prefix all cmds with nmd.sh poop poop poop
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
