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
- requires you have a [local docker registry setup](../docker/README.md)
- access the UI @ [https://mad.nirv.ai:4646](https://mad.nirv.ai:4646/ui/jobs)
- requires you've setup [TLS with cloudflare cfssl](https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls)

#### helpful links

- [error about cni plugin](https://discuss.hashicorp.com/t/failed-to-find-plugin-bridge-in-path/3095)
- [enabling bind mounts](https://developer.hashicorp.com/nomad/docs/drivers/docker#enabled-1)
- [vault container requires ipc_lock](https://developer.hashicorp.com/nomad/docs/drivers/docker#allow_caps)

#### REQUIREMENTS

- [you have nirvai/scripts in your path](../scripts/README.md)
- your directory structure looks like this

```sh
./you-are-here
├── configs # TODO: put development nomad configs here
├── core # your monorepo
    ├── apps
        ├── nirvai-core-nomad
            ├── dev
                ├── tls
                    ├── cfssl.json
                    ├── cli.csr
                    ├── client.csr
                    ├── client-key.pem
                    ├── client.pem
                    ├── cli-key.pem
                    ├── cli.pem
                    ├── nomad-ca.csr
                    ├── nomad-ca-key.pem
                    ├── nomad-ca.pem
                    ├── server.csr
                    ├── server-key.pem
                    └── server.pem
                ├── development.client.nomad
                ├── development.dev_core.nomad
                ├── development.server.nomad
                ├── .env.development.compose.(json|yaml)
├── scripts

```

#### INTERFACE

```sh
# script.nmd.sh requires the following vars
NOMAD_ADDR_SUBD=${ENV:-dev}
NOMAD_ADDR_HOST=${NOMAD_ADDR_HOST:-nirv.ai}
NOMAD_SERVER_PORT="${NOMAD_SERVER_PORT:-4646}"
NOMAD_ADDR="${NOMAD_ADDR:-https://${NOMAD_ADDR_SUBD}.${NOMAD_ADDR_HOST}:${NOMAD_SERVER_PORT}}"
NOMAD_CACERT="${NOMAD_CACERT:-./tls/nomad-ca.pem}"
NOMAD_CLIENT_CERT="${NOMAD_CLIENT_CERT:-./tls/cli.pem}"
NOMAD_CLIENT_KEY="${NOMAD_CLIENT_KEY:-./tls/cli-key.pem}"

```

#### basic workflow

```sh
########### cd nirvai/core
# refresh development compose containers and upsert .env.${ENV}.compose.{json,yaml}
script.refresh.compose.sh


# ensure you've completed steps in registry.sh (see above)
# start the registry
script.registry.sh run
# tag-and-push all images backing running containers to the registry
script.registry.sh tag_running
# stop all development containers (nomad uses the images in the registry)
docker compose down


########### cd ./apps/nirvai-core-nomad/dev
# symlink the json & yaml development variables to nomad dev dir
ln -s ../../../.env.development.compose.* .

# start server agent in bg
script.nmd.sh start s -config=development.server.nomad
# start client agent in bg
script.nmd.sh start c -config=development.client.nomad
# check the team status
script.nmd.sh get status team


# if ./development.dev_core.nomad doesnt exist
# create it and get the index number for stdout
script.nmd.sh get plan dev_core

# deploy job dev_core
script.nmd.sh run dev_core indexNumber
script.nmd.sh dockerlogs # [optional] see logs of all running containers

# on error run
script.nmd.sh get status job jobName # includes all allocationIds
script.nmd.sh get status loc allocationId # in event of deployment failure
script.nmd.sh get status node # see client nodes and there ids
script.nmd.sh dockerlogs # following docker logs of all running containers
nomad alloc exec -i -t -task sidekiq fa2b2ed6 /bin/bash # todo,
nomad alloc exec -i -t -task puma fa2b2ed6 /bin/bash -c "bundle exec rails c" #todo
nomad job history -p job_name # todo

# cleanup
# rm the job
script.nmd.sh rm dev_core
# kill the team @see https://github.com/noahehall/theBookOfNoah/blob/master/linux/bash_cli_fns/000util.sh
kill_service_by_name nomad
# reset nomad to a green state if you dont plan on using it later
nomad system gc

###################### USAGE
## prefix all cmds with script.nmd.sh poop poop poop
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
