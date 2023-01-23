# NIRVai Nomad docs

- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/nomad)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/nomad)

## RACEXP

- [NIRV DOCS IaC board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fiac%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## NIRVai Mindset

> if it doesnt copypasta, it doesnt belong in your stack

insert video here

## WHY NOMAD ?

- put stuff here

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

- copypasta from consul, we need something specific for nomad

> all services must follow [PoLP](https://www.upguard.com/blog/principle-of-least-privilege) and require authnz

## validation: transition from local to cloud services

- transitioning from: active development with docker compose
  - i.e. where:
    - apps are developed
    - unit tests are accepted
- transitioning to: validation with nomad orchestration
  - i.e. where
    - integration & e2e tests are accepted
    - security controls established and first round of obfuscation occurs
    - service runtimes mirror prod infrastructure

### helpful links

- [error about cni plugin](https://discuss.hashicorp.com/t/failed-to-find-plugin-bridge-in-path/3095)
- [enabling bind mounts](https://developer.hashicorp.com/nomad/docs/drivers/docker#enabled-1)
- [vault container requires ipc_lock](https://developer.hashicorp.com/nomad/docs/drivers/docker#allow_caps)

### REQUIREMENTS

- [complete CFSSL setup](../cfssl/README.md)
  - be sure to create a p12 cert to access nomad UI from your browser
- [see the env docs for how to set up /etc/ssl/certs](../env/README.md)
- your services are ready for deployment

```sh
# jq:       # @see https://stedolan.github.io/jq/manual/
# nomad:    # @see https://developer.hashicorp.com/nomad/docs/install

# directory structure matches:
├── scripts      # @see https://github.com/nirv-ai/scripts
├── configs      # @see https://github.com/nirv-ai/configs
├── $REPO_DIR_NAME
    └── iac/$ENV                  # validation|test|stage|production
        ├── $REPO_DIR_NAME.nomad  # jobspec for this stack
        ├── client.nomad          # nomad client conf
        ├── server.nomad          # nomad server conf

```

### INTERFACE

```sh
###########

NOMAD_ADDR_SUBD=dev
NOMAD_ADDR_HOST=nirv.ai
NOMAD_SERVER_PORT=4646
NOMAD_ADDR=https://$NOMAD_ADDR_SUBD.$NOMAD_ADDR_HOST:$NOMAD_SERVER_PORT
NOMAD_CACERT=/etc/ssl/certs/mad.nirv.ai/ca.pem
NOMAD_CLIENT_CERT=/etc/ssl/certs/mad.nirv.ai/cli-0.pem
NOMAD_CLIENT_KEY=/etc/ssl/certs/mad.nirv.ai/cli-0-key.pem

```

### start nomad server and client agents

```sh
# prefix all cmds with script.nmd.sh
###########
# start server agent in bg
start server

# start client agent in bg
start client

# check the server status
get status servers

# check the client status
get status clients
# open the Nomad UI: https://mad.nirv.ai:4646
```

### deploy nomad jobspec

```sh
create stack core

## get a fresh job plan and retrieve the index number from stdout
## we validate every config and jobspec, deal with the errors
get plan core

## deploy the core stack
run core indexNumber

## review logs of containers initialized by nomad
dockerlogs

## on error run
get status job jobName # includes all allocationIds
get status loc allocationId # in event of deployment failure
get status clients # see client nodes and there ids
dockerlogs # following docker logs of all running containers
nomad alloc exec -i -t -task sidekiq fa2b2ed6 /bin/bash # todo,
nomad alloc exec -i -t -task puma fa2b2ed6 /bin/bash -c "bundle exec rails c" #todo
nomad job history -p job_name # todo

## cleanup
## rm the job
rm core

## requires shell-init
kill

## reset nomad to a green state if you dont plan on using it later
gc
```

## next steps

- Congrats!
- checkout usage [usage docs](./usage.md)
