# NIRVai VAULT

- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/vault)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/vault)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## NIRVai Mindset

> if it doesnt copypasta, it doesnt belong in your stack

https://user-images.githubusercontent.com/10324554/212817830-3cfe7012-d408-4dab-af08-aace1763dfd0.mp4

## WHY VAULT ?

- vault is an opensource database for sensitive data storage with dynamic TTL authnz for a broad set of stores and cloud services
- our goal is to achieve:
  - zero trust
  - complete immutable infrastructure from dev to prod
  - single source of truth for data access, control and expiration
  - automated east-west credential provisioning
  - developer empowerment with operator oversight

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

> if you cant commit it - _vault it_

### if your application has network access

- remove all your `.env` and other configuration files from your `.gitignore`
- move all your sensitive data into vault
- refactor your application to be resiliant despite unmet requirements
- ensure your application uses a modern TLS version for all communication
- ensure your application encrypts data in transit
  - if persisting sensitive data to disk, dont, fkn vault it
- commit your remaining data & configuration to git (which should contain no PII/etc)

### if your application is `air-gapped`

- ensure your application runs from a [chroot](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/)
- continue with [12-factor](https://12factor.net/)

### How do I know if this data is sensitive?

- it is

## Setting up vault

- generally this should be the first step _in all microservices_ deployed at NIRVai
- `unless` your application requires access to 0 sensitive data and relies purely on configuration committed to git
- config/core vs config/$APP_NAME & $REPO_DIR
  - config/core: owned by vault operators
    - the core directory should be the single source of truth and under tight security
  - config/$APP_NAME: vault configuration files for a downstream app approved by operators
  - $REPO_DIR: the vault application being bootstrapped: could be config/core or config/$APP_NAME
    - useful for migrating an app to zero trust while offloading app integration & testing to app owners
    - when integration is complete
      - operators can validate their vault configs: then copy those files into an isolated config/$APP_NAME DIR
      - the core vault server can then consume those isolated files with 0 headaches
      - app owners can continue to iterate and repeat the request > approve > cycle

### REQUIREMENTS

> if your directory structure _does not_ match below
> modify the inputs in the `# INTERFACE` section that follows

```sh

######################### REQUIREMENTS
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/
# debian compatible host (only tested on ubuntu)

# directory structure matches:
├── scripts             # @see https://github.com/nirv-ai/scripts
├── configs             # @see https://github.com/nirv-ai/configs
│   └── vault
│   │   └── core        # init files for the upstream vault server
│   │   └── ${APP_NAME} # isolated init files for an arbitrary app
                        # core can also contain any of the following (in initiatilization order)
│   │   │   ├── vault-admin/*      # super user policy
│   │   │   ├── policy/*           # non super user and machine policies
│   │   │   ├── token-role/*       # token role policies
│   │   │   ├── enable-feature/*   # enabled vault features
│   │   │   ├── auth/*             # auth & app role configuration
│   │   │   ├── secret-engine/*    # secret engine(s) configuration
│   │   │   ├── token/*            # hydrate default tokens
│   │   │   ├── secret-data/*      # hydrate secret engines
├── ${REPO_DIR_NAME}
│   └── apps/$APP_PREFIX-$APP_NAME/src/vault/config/*
├── secrets             # chroot jail, a temporary folder or private git repo
│   └── vault
│   │   └── tokens
│   │   │   ├── root/*  # directory for root and unseal tokens
│   │   │   ├── admin/* # directory for admin token(s)
│   │   │   ├── other/* # directory for non-admin and machine token(s)


```

### INTERFACE

- all green field vault instances require human intervention
  - create root pgp key
  - create root token and unseal database
  - create and distribute admin token(s)
  - store root token next to your grenade launchers

```sh
######################### platform interface
# all nirv scripts use this same interface
export APP_DIR_NAME=apps
export APP_PREFIX=nirvai
export CONFIG_DIR_NAME=configs
export JAIL_DIR_NAME=secrets
export NIRV_SCRIPT_DEBUG=0
export REPO_DIR_NAME=core


######################### VAULT INTERFACE
# we only support secured vault servers
export VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8201
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"

# set to empty string if you dont have a token yet
export VAULT_TOKEN=''

# set the current admin name
export VAULT_ADMIN_NAME='admin'

# the vault server your bootsrapping
export VAULT_APP_SRC_PATH='src/vault'
export VAULT_SERVER_APP_NAME='core-vault'

# set to a specific app (e.g. core or $APP_NAME)
# or leave as empty string to bootstrap all vault config files
export APP_TARGET=''

```

### NEW VAULT SERVER SETUP

#### create root and admin_vault pgp keys

```sh
######################### HUMAN REQUIRED
# a human is required to create and distribute
# root & admin vault tokens and unseal keys
# @see https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key

## create root pgp key with optins: 1 > 4096 > 0 > y
## list the key: will output e.g. sec rsa4096/3AA61DC439A44276 2023-01-16 [SC]
## then save it as in asc format
## everything saved in $JAIL/tokens/root/root.{asc,gpg}
script.vault.sh create gpg root-key
gpg --list-secret-keys --keyid-format=long
script.vault.sh create gpg root-asc 3AA61DC439A44276


## repeat the process for yourself (default: admin)
## everything saved in $JAIL/tokens/admin/$VAULT_ADMIN_NAME.{asc,gpg}
script.vault.sh create gpg admin-key
gpg --list-secret-keys --keyid-format=long
script.vault.sh create gpg admin-asc 3AA61DC439A44276

## repeat for as many admins you need

### hypothetical token for CTO
## everything saved in $JAIL/tokens/admin/armon.{asc,gpg}
script.vault.sh create gpg admin-key armon
gpg --list-secret-keys --keyid-format=long
script.vault.sh create gpg admin-asc 3AA61DC439A44276 armon

### hypothetical token for CISO
## everything saved in $JAIL/tokens/admin/mitchell.{asc,gpg}
script.vault.sh create gpg admin-key mitchell
gpg --list-secret-keys --keyid-format=long
script.vault.sh create gpg admin-asc 3AA61DC439A44276 mitchell
```

#### wipe and initialize vault server

```sh
## stop and remove any running containers
# if you added the shell-init scripts, you can run
dk_stop_rm_cunts

# remove previous vault data {e.g. raft,vault}.db
script.vault.sh rm vault-data

# forcefully sync configs/vault to app_dir/vault
script.vault.sh sync-confs

# finally: cd to your repo dir and start your entire stack
script.reset.compose.sh

######################### initialize vault
export VAULT_TOKEN='initializing vault'

## verify response.initialized = false & sealed = true
script.vault.sh get status

## inititialize vault with root and *all* created admin tokens, and create unseal tokens
## unseal tokens saved as $JAIL/tokens/root/unseal_tokens.json
## you can optionally pass a threshold number (default is 2)
script.vault.sh init


# verify response.initialized = true & sealed = true
script.vault.sh get status

```

#### set root token and unseal db

```sh
## export root token
export NIRV_SCRIPT_DEBUG=0 # must be disabled for retrieving tokens
export VAULT_TOKEN=$(script.vault.sh get token root)

## (requires password used when creating the pgp key)
script.vault.sh unseal
script.vault.sh get status
script.vault.sh get token self
```

#### use root token to create admin policy & token(s)

> this is the last time you should ever use the root token

```sh
# create policies and tokens for all admins
# this requires you to setup the token configuration for each (see configs)
# tokens saved in $JAIL_VAULT_ADMIN
script.vault.sh process vault_admin
```

#### set admin token and unseal db

```sh
## use any admin token to authenticate to vault
export NIRV_SCRIPT_DEBUG=0 # must be disabled for retrieving tokens
export VAULT_ADMIN_NAME=admin
export VAULT_TOKEN=$(script.vault.sh get token admin)

## unseal the DB if its sealed and verify vault server status & token info
## (requires password used when creating the pgp key)
script.vault.sh unseal
script.vault.sh get status
script.vault.sh get token self
```

#### use admin token to create policies in policy dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

script.vault.sh process policy
```

#### use admin token to create token roles in token role dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

script.vault.sh process token_role
```

#### use admin token to enable features in feature dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# filename template: enable.THIS_THING.AT_THIS_PATH
## e.g. enable.kv-v2.secret # for versioned data
## e.g. enable.kv-v1.env # for immutable data with increased perf
## currently we only auto enable features at top-level paths
### wont work: enable.kv-v2.microfrontend.app1.snazzle
### will work: enable.kv-v2.mfe-app1-snazzle

# enable all features in feature dir
script.vault.sh process enable_feature
```

#### use admin token to configure auth schemes in auth dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# filename syntax: auth_approle_role_ROLE_NAME.json
## e.g. auth_approle_role_backend.json
## e.g. auth_approle_role_kafka.json

script.vault.sh process auth
```

#### use admin token to configure secret engines in engine dir

> this requires a running DB
> on subsequent runs you can ignore the db init errors (vault user creds already rotated)

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# depending on the type of secret engine your configuring
# the filename template will have different formats:

## kv-v2: secret_kv2.PATH.config.json
### ^ configure the kv2 engine enabled at PATH/

## database config: secret_database.DB_NAME.config.json
### ^ configure the database DB_NAME

## database role config: secret_database.DB_NAME.role.ROLE_NAME.json
### ^ configure ROLE_NAME for database DB_NAME
### ^ we only support databases that support rotate-root creds
### ^ the vault root creds will be automatically rotated

script.vault.sh process secret_engine
```

#### use admin token to create initial auth tokens for downstream services

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# depending on the type of authentication scheme
# the filename template will have different formats:

## app role template: token_create_approle.ROLE_NAME.FILE_NAME
## e.g. token_create_approle.auth_approle_role_bff.bff
## ^ save role-id and secret-id as $JAIL/other/auth_approle_role_bff.{bff,id}.json

## token role template: token_create_token_role.ROLE_NAME.FILE_NAME
## e.g. token_create_token_role.batch_infra.cd
## ^ save batch token for role batch_infra as $JAIL/batch_infra.cd.json
#### by default batch tokens expire after 20 minutes: good for CD
## e.g. token_create_token_role.periodic_infra.ci
## ^ save periodic token for role periodic_infra as $JAIL/periodic_infra.ci.json
#### by default periodic must renew within 30 days: good for CI

script.vault.sh process token
```

#### use admin token to hydrate initial secret data for downstream services

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# depending on the type of secret engine
# the filename template will have different formats:

## KV V1 secret engine: hydrate_kv1.ENABLED_AT_THIS_PATH.STORE_SECRET_AT_THIS_PATH
## e.g. secret_kv1.env.auth_approle_role_bff.json
## ^ upsert data at secret-path auth_approle_role_bff in the kv1 engine enabled at env

## KV V2 secret engine: hydrate_kv2.ENABLED_AT_THIS_PATH.STORE_SECRET_AT_THIS_PATH
## e.g. secret_kv2.secret.auth_approle_role_bff.json
## ^ upsert data at secret-path auth_approle_role_bff in the kv2 engine enabled at secret
script.vault.sh process secret_data
```

#### next steps

- Congrats! you have a zero trust development environment backed by VAULT!
- If you want to do more with vault: [checkout the usage documentation](./usage.md)
- We have a little secret:

> _you can bootstrap your entire stack with this copypasta_

##### copypasta: vault initialization

```sh
######################### FYI
# a human is required to create the initial root and admin tokens
# before continuing: complete `create root and admin_vault tokens`

export APP_DIR_NAME=apps
export APP_PREFIX=nirvai
export CONFIG_DIR_NAME=configs
export JAIL_DIR_NAME=secrets
export NIRV_SCRIPT_DEBUG=0
export REPO_DIR_NAME=core
export VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8201
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"
export VAULT_TOKEN=''
export VAULT_ADMIN_NAME='admin'
export VAULT_APP_SRC_PATH='src/vault'
export VAULT_SERVER_APP_NAME='core-vault'

########## START: vault boot type
# stop and remove running containers
dk_stop_rm_cunts

# sync app confs with the source of truth
script.vault.sh sync-confs


########## how are you booting your devstack?
# you need to be wherever your compose.yaml file is
cd $REPO_DIR_NAME

## default: wipe system and start from scratch
dk_rm_all
script.vault.sh rm vault-data
script.reset.compose.sh

# it takes a few seconds for services to register with consul
sleep 1s

## alternative 1: wipe vault, but keep everything else
# script.vault.sh rm vault-data
# script.refresh.compose.sh

## DEFAULT & ALTERNATIVE 1
## required: vault must be initialized
export VAULT_TOKEN='initializing vault'
script.vault.sh init

## required: root must create atleast 1 admin token
export VAULT_TOKEN=$(script.vault.sh get token root)
script.vault.sh unseal
script.vault.sh process vault_admin

## alternative 2a: simple reboot
# script.refresh.compose.sh

########## your devstack is now running! but we need a database
# for demonstrative purposes, lets use a DB in a totally different project
cd ../web
script.reset.compose.sh web_postgres 1

# connect vault to the webnetwork so they can chat
docker network connect webnetwork nirvai-core-vault-1

########## time to bootstrap vault

## login as admin & unseal vault
export VAULT_TOKEN=$(script.vault.sh get token admin)
script.vault.sh unseal

## (re)sync vault data if changed/starting from scratch
script.vault.sh process policy
script.vault.sh process token_role
script.vault.sh process enable_feature
script.vault.sh process auth
script.vault.sh process secret_engine
script.vault.sh process token
script.vault.sh process secret_data

```

##### copypasta: validation

```sh
# validate admin token
echo -e "\n\nwhoami"
script.vault.sh get status
script.vault.sh get token self
script.vault.sh get_unseal_tokens

# validate dynamic db creds
echo -e "\n\ndb creds"
script.vault.sh get postgres creds readonly
script.vault.sh get postgres creds readwrite
script.vault.sh get postgres creds readstatic

# validate engines, approles and secrets
echo -e "\n\nsecret engines"
script.vault.sh list secret-engines | jq 'keys'
echo -e "\n\napproles"
script.vault.sh list approles
echo -e "\n\nsecret kv1 keys"
script.vault.sh list secret-keys kv1 | jq '.data.keys'
echo -e "\n\nsecret kv2 keys"
script.vault.sh list secret-keys kv2 | jq '.data.keys'

# now dont forget! delete your shell history
history -c
```

#### Log into the UI

```sh
## the below steps require completion of:
## section: `create root and admin_vault tokens`
## section: `use root token to create admin policy & token`

## copypasta the tokens and open the browser to $VAULT_ADDR
## authenticating a VAULT_ADDR initialized with a different root token is prohibited
## requires clearing your browser storage and cache

############################################
################# DANGER ###################
## this logs all your unseal tokens & vault token as plain text in your shell
## this logs the minimum amount of unseal tokens required to unseal vault
script.vault.sh get_unseal_tokens

## or you can retrieve a single unseal token only (wont log your vault token)
script.vault.sh get_single_unseal_token 0 # or 1 for the second, or 2 for third, etc.
################# DANGER ###################
############################################
```
