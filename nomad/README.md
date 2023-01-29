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

## create a fresh job plan and retrieve the index number from stdout
## we validate every config and jobspec, deal with the errors
create plan core

## deploy the core stack
run core indexNumber

## cleanup
## rm/stop the job
rm core
stop core

## requires shell-init
kill

## reset nomad to a green state if you dont plan on using it later
gc
```

## usage

- TODO: move this entire section to usage.md

```sh

## review logs of running containers
## TODO: move this to one of the docker scripts
dockerlogs
dockerlogs-kill # cleanup when finished

## inspection
get client [ID]
get dep [DEPLOYMENT_ID] # all/specific deployment
get eval [EVAL_ID] # all/specific evaluation
get loc [ALLOCATION_ID] # get all/specific allocation info
get self # info about local nomad agent
get server
get service [SRV_NAME] # list all/specific services
get stack [STACK_ID] # all/specific stack (jobs)


## these wont work anymore unless you set the CA_CERT related vars
## as we've set verify true to everything
## we should add these to script.nmd.sh
nomad alloc exec -i -t -task sidekiq fa2b2ed6 /bin/bash # todo,
nomad alloc exec -i -t -task puma fa2b2ed6 /bin/bash -c "bundle exec rails c" #todo
nomad job history -p job_name # todo

```

## next steps

- Congrats!
- checkout usage [usage docs](./usage.md)
  - TODO: this file is seriously out of date
