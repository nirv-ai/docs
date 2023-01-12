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

### USAGE

```sh

```
