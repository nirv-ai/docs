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
  - north-south authN service discovery (haproxy > envoy > consul > envoy > app)
  - east-west authN+Z service mesh (app > envoy > consul > envoy > app)
  - dynamic service configuration (consul-template, envconsul, cli, http)

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

> all services must follow [PoLP](https://www.upguard.com/blog/principle-of-least-privilege) and require authnz

## Setting up CONSUL

### REQUIREMENTS

- [complete CFSSL setup](../cfssl/README.md)

```sh
# jq: @see https://stedolan.github.io/jq/manual/

# directory structure matches:
├── scripts         # @see https://github.com/nirv-ai/scripts
├── configs         # @see https://github.com/nirv-ai/configs
│   └── consul
│   │   └── config  #
│   │   │   ├── poop.soup.boop #
│   │   └── host    #
│   │   │   ├── poop.soup.boop #
│   │   └── policy  #
│   │   │   ├── poop.soup.boop #
│   │   └── service #
│   │   │   ├── poop.soup.boop #
│   │   ├── cfssl.json #
│   │   │   ├── poop.soup.boop #
├── secrets         # chroot jail, a temporary folder or private git repo
│   └── arbitrary.domain.com
│   │   └── tls     # should contain all required tls certs
```

### INTERFACE

```sh
# you generally only need to set the CA_CN var when your directory structure matches
export CA_CN=mesh.nirv.ai

# else you can map the these values to the directory structure above


# you shouldnt (but can) change these as well

```

### service mesh workflow

```sh
################ cd nirv
# TODO: move this into docs and fkn automate this shiz like script.vault.sh
# ensure consul:consul is setup on host
# ensure source (not symlinks) files & secrets on host are owned by consul:consul
# see dockerfiles for how to force img consul:consul to match host consul:consul
### create rootca & server certs
# create tokens rootca, server client & cli certs using script.ssl.sh
# ^ make sure to create server certs for each service
# ^ it wont overrwrite existing certs, so just manually run the server create with X total
# ^^ pretty sure services should receive client certs and not server certs
# ^^ TODO: fix this logic to start from x+1 based on existing files in dir with same name
# ^^ scratch that, as we will be moving to vault PKI eventually anyway
# `create gossipkey` > copy from jail to core-consul
# delete data/* if starting green
# `sync-confs` push configs > server & service(s) app directories
# script.reset core-consul
# `source configs/consul/.env.cli
# `get info` >>> ACL NOT FOUND
# `create root-token` >>> docker log: bootstrap complete
# `source configs/consul/.env.cli` # wont set correct values if debugging is on
# `get root-token` >>> validate UI login
# should have access to almost everything
### create policy files and tokens and push to consul server
# see config policy dir
# `create policies`
# `create server-policy-tokens`
# `create service-policy-tokens` # service names must match svc configs
# `create intentions`
# `create defaults`
# `list policies`
# `list tokens`
# `set server-tokens` >>> docker log from [WARN] agent: ...blocked by acls... --> to agent: synced node info
# ^ just to get through initial setup
# ^ check how we set the servers DNS_TOKEN and HTTP_TOKEN via bootstrap.sh and /.env
# ^^ TODO: automate setting .env of all consul containers to appropriate tokens
# `get nodes` > every node should have a taggedAddress, else ACLs/tokens/wtf arent setup properely
# exec into any consul enabled container, and confirm  `consul members`
### ADR: this is an ideal use-case for multi-app containers
# update docker images to include binary (see proxy for ubuntu, vault for alpine)
# `sync-confs` push configs > to server & service(s) app dirs
# in the docker .env for all serv[ers,ices]: set vars HTTP/DNS tokens vars, see bootstrap.sh files
# ^ set CONSUL_HTTP_TOKEN=$(script.consul.sh get service-token svc-name) # TODO: move to docker secret
# ^ TODO: create a token specifically for connect and dont reuse the same agent token
# ^ TODO: ^ also need modify the policy for agent tokens follow least privileges
# ^ TODO: or follow consul guidance and create specific intention tokens that are given to admins
# ^ CONNECT_SIDECAR_FOR=svc-name
# sudo rm -rf app/svc-name/src/consul/data/* if starting from scratch
# script.reset.sh
# ^ `get team` >>> w00p w00p
# ^ `get nodes` >>> w00p w00p
#### haproxy north-south + consul dns

```

### USAGE

```sh
### prefix all cmds with script.consul.sh

```
