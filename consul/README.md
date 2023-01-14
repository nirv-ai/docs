# NIRVai CONSUL

- [scripting architecture & guidance](.scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/consul/script.consul.sh)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/consul)
- [based heavily on these scripts by hashicorp](https://github.com/hashicorp-education/learn-consul-get-started-vms/tree/main/scripts)

## RACEXP

- [NIRVai networking project](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fnetworking%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## WHY CONSUL ?

## Setting up CONSUL

### REQUIREMENTS

```sh
# complete CFSSL documentation: @see https://github.com/nirv-ai/docs/blob/main/cfssl/README.md
# jq: @see https://stedolan.github.io/jq/manual/

# directory structure matches:
├── scripts # git clone git@github.com:nirv-ai/scripts.git
├── configs # you can use ours: git clone git@github.com:nirv-ai/configs.git
│   └── consul
│   │   └── config #
│   │   │   ├── poop.soup.boop #
│   │   └── host #
│   │   │   ├── poop.soup.boop #
│   │   └── policy #
│   │   │   ├── poop.soup.boop #
│   │   └── service #
│   │   │   ├── poop.soup.boop #
│   │   ├── cfssl.json #
│   │   │   ├── poop.soup.boop #
├── secrets # chroot jail, a temporary folder or private git repo
│   └── arbitrary.domain.com
│   │   └── tls # should contain all required tls certs
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
# copy config/server/* to core-consul
# `create gossipkey` > copy from jail to core-consul
# delete data/* if starting green
# script.reset core-consul
# `get info` >>> Error querying agent: Unexpected response code: 403 (Permission denied: token with AccessorID '00000000-0000-0000-0000-000000000002' lacks permission 'agent:read' on "consul")
# `create root-token`
# `source configs/consul/host/.env.cli` # wont set correct values if debugging is on
# `get info`
# `get root-token` >>> validate UI login
# should have access to almost everything
### create policy files and tokens and push to consul server
# see config policy dir
# `create policies`
# `list policies`
# `create server-policy-tokens`
# `create service-policy-tokens` # service names must match svc configs
# `source configs/consul/host/.env.consul.server`
# `list tokens`
# `set server-tokens` >>> from [WARN] agent: ...blocked by acls... --> to agent: synced node info
# `get nodes` > every node should have a taggedAddress, else ACLs/tokens/wtf arent setup properely
### update docker images to include binary (see proxy for ubuntu, vault for alpine)
### DISCOVERY: add configs for to each client machine
# @see https://developer.hashicorp.com/consul/tutorials/get-started-vms/virtual-machine-gs-service-discovery
# create a base discovery/client/config/* that can be used as defaults for each specific client service
# create discovery/service-name/config/* configs
# copy discovery/{client,service-name}/configs/* into each app/service-name/consul/src/config
# copy secrets/gossip to each as well
# validate each config has the data it needs
# ^ make sure to (TODO) manually set token values from secrets/consul/token
# ^ `get team` >>> w00p w00p
# ^ `get nodes` >>> w00p w00p
# sudo rm -rf app/svc-name/src/consul/data/* if starting from scratch
# script.reset|refresh compose_service_name(s) to boot consul clients
### MESH: this is a migration from discovery to mesh


```

### USAGE

```sh

```
