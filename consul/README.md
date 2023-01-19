# NIRVai CONSUL

- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/consul)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/consul)
- [based heavily on these scripts by hashicorp](https://github.com/hashicorp-education/learn-consul-get-started-vms/tree/main/scripts)

## RACEXP

- [NIRVai networking project](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fnetworking%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## NIRVai Mindset

> if it doesnt copypasta, it doesnt belong in your stack

insert video here

## WHY CONSUL ?

- consul, in its final form, is an authnz service mesh with support for dynamic configuration (kv) management and first class envoyproxy integration for east-west/north-south traffic, and second class (but well documented) support for haproxy
- we are happy with the HAProxy for north-south traffic, and it can be just as dynamic as envoy, but lacks the intuitive interface consul-envoy provides even considering haproxy's first class consul support via dataplane api
- Our goal is to achieve:
  - zero trust
  - complete immutable infrastructure from dev to prod
  - north-south authN+Z service discovery (haproxy > envoy > consul > envoy > app)
  - east-west authN+Z service mesh (app > envoy > consul > envoy > app)
  - dynamic service configuration (consul-template, cli, http)
  - multi-app containers: consul and envoy should be baked into the service image

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

> all services must follow [PoLP](https://www.upguard.com/blog/principle-of-least-privilege) and require authnz

## Setting up consul

### REQUIREMENTS

```sh
# jq:       # @see https://stedolan.github.io/jq/manual/
# consul:   # @see https://developer.hashicorp.com/consul/docs/install

# directory structure matches:
├── scripts                     # @see https://github.com/nirv-ai/scripts
├── configs                     # @see https://github.com/nirv-ai/configs
│   └── consul
│   │   └── client              # client agent confs
│   │   └── defaults            # default confs for client services
│   │   └── global              # confs applied to both server & client agents
│   │   └── intention           # service authZ confs
│   │   └── policy
│   │   │   └── server          # acl policies for server agents
│   │   │   └── service         # acl policiess for services
│   │   └── server              # server agent confs
│   │   └── service/$APP_X..Y
│   │   │   └── config          # confs for this consul service
│   │   │   └── envoy           # static envoy confs
│   │   ├── .env.cli            # source this file to authnz with a consul server from host
├── secrets                     # chroot jail, a temporary folder or private git repo
│   └── consul
│   │   └── keys                # we will persist created keys to this directory
│   │   └── tokens              # we will persist created tokens to this directory
├── $REPO_DIR_NAME
│   └── apps/$APP_PREFIX-$APP_X..Y
│   │   ├── $APP_ENV_AUTO                         # non empty file where vars are injected
│   │   │   ├── CONSUL_HTTP_TOKEN                 # consul agent token
│   │   │   ├── CONSUL_NODE_PREFIX                # for setting -node=THIS_NAME-$(hostname)
│   │   │   ├── CONSUL_DNS_TOKEN                  # set as the default token for servers
│   │   └── src/consul
│   │   │   │   ├── consul.compose.bootstrap.sh   # runtime init for consul & envoy
│   │   │   └── config
│   │   │   │   ├── config.client.*               # client only confs
│   │   │   │   ├── config.global.*               # server & clients confs
│   │   │   │   ├── config.server.*               # server only confs
│   │   │   │   ├── config.service.$APP_X..Y      # service specific confs
│   │   │   │   ├── env.token.hcl                 # created by bootstrap.sh: agent & default token
```

### INTERFACE

```sh
######################### platform interface
# all nirv scripts use this same interface
export APP_DIR_NAME=apps
export APP_ENV_AUTO=.env.auto
export APP_PREFIX=nirvai
export CERTS_DIR_HOST=/etc/ssl/certs
export MESH_HOSTNAME=mesh.nirv.ai
export NIRV_SCRIPT_DEBUG=0
export REPO_DIR_NAME=core

######################### CONSUL INTERFACE
# path within the app dir
export CONSUL_APP_SRC_PATH='src/consul'

# e.g. /etc/ssl/certs/mesh.nirv.ai
export CONSUL_DIR_CERTS="${CERTS_DIR_HOST}/${MESH_HOSTNAME}"
export CONSUL_SERVER_APP_NAME='core-consul'
export CONSUL_SERVER_NODE_PREFIX='consul'
export DATA_CENTER='us-east'
export DNS_TOKEN_NAME='acl-policy-dns'
export ROOT_TOKEN_NAME='root'
export SERVER_TOKEN_NAME='acl-policy-consul'
```

### NEW SERVICE MESH SETUP

- [complete CFSSL setup](../cfssl/README.md)
  - [see the env docs for how to set up /etc/ssl/certs](../env/README.md)
- if using our one of our [docker configs](https://github.com/nirv-ai/configs/tree/develop/docker)
  - take a look at core/{consul/vault/haproxy}
    - bootstrap.sh files: varies depending if initiating a consul server or client and type of application
    - compose.yaml: we inject the TLS certs as secrets into the container

```sh
## > see bootstrap.sh files for clients
# need to set token specifically for connect
# ^ else the default is used for envoy
# ^ this new token should be set in src/.env.auto
# need to ensure no one uses the management token
# ^ create a admin tokens like in vault
# TODO: create a token specifically for connect and dont reuse the same agent token
# TODO: ^ also need modify the policy for agent tokens follow least privileges
# TODO: or follow consul guidance and create specific intention tokens that are given to admins

```

#### create gossip key & root token, sync confs and start your stack

```sh
################
# cd to your app dir
cd $REPO_DIR_NAME


################ delete data and existing tokens
## TODO: move these to consul.sh
sudo rm -rf apps/*/src/consul/data/*
sudo rm -rf ../secrets/consul/{tokens,keys}/*


################# create gossipkey, sync confs and start your stack
## create gossip key
script.consul.sh create gossipkey
script.consul.sh sync-confs
script.reset.compose.sh # >>> docker logs: blocked by ACLS


################# create root token and setup your cli
## copy and execute the cmd thats output then create a root token
## this is a manual step: you have to copy paste the output
script.consul.sh get cli-env
script.consul.sh create root-token # >>> docker log: bootstrap complete

## re-execute the cmd output by cli-env
## this is a manual step: you have to copy paste the output
## the consul server(s) should be the only member on the team
script.consul.sh get cli-env
```

#### use root token to create initial service mesh config

```sh
################# create policies, tokens, intentions and service defaults
script.consul.sh create policies
script.consul.sh create server-policy-tokens
## FYI: service names must match svc configs
script.consul.sh create service-policy-tokens
script.consul.sh create intentions
script.consul.sh create defaults


################# review created policies and tokens
script.consul.sh list policies
script.consul.sh list tokens


################# update .env.auto, resync confs and restart stack
script.consul.sh sync-env-auto
script.consul.sh sync-confs
script.reset.compose.sh

# every node should have a taggedAddress, else ACLs/tokens/wtf arent setup properely
## >>> docker log: agent: synced node info
script.consul.sh get nodes
script.consul.sh get team

```

#### logging into the UI

```sh
script.consul.sh get root-token

```

### USAGE

```sh
### prefix all cmds with script.consul.sh

```

#### next steps

- Congrats! you have a zero trust sservice mesh backed by CONSUL!
- We have a little secret:

> _you can bootstrap your entire stack with this copypasta_

##### copypasta: consul initialization

```sh
# create gossip key
# sync confs
# delete data/* if starting green

```
