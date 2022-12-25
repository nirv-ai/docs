# nirvai VAULT

- documentation for VAULT @ nirvai

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## WHY VAULT ?

- vault is an opensource database for sensitive data storage, access and control

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

> if you cant commit it, vault it

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

```sh

######################### FYI
# in all examples: /chroot/jail = ../secrets/dev/apps/vault
# @see https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/

# havent had any success in curling hcl to vault http api, convert to json via this tool
# @see https://www.convertsimple.com/convert-hcl-to-json/
# haha ^ didnt work either
# fear the copypasta: https://gist.github.com/v6/f4683336eb1c4a6a98a0f3cf21e62df2

######################### interface
# wherever you will temporarily store created secrets on disk
export JAIL="../secrets/dev/apps/vault"
# address where you expect the vault server to be running
export VAULT_ADDR=https://dev.nirv.ai:8300
# config directory for the vault server instance you are bootstrapping
export VAULT_INSTANCE_DIR=apps/nirvai-core-vault/src


# unseal cmd
## you will need to rerun this whenever you restart the vault server
unseal_threshold=$(cat $JAIL/root.unseal.json | jq '.unseal_threshold')
i=0
while [ $i -lt $unseal_threshold ]; do
    vault operator unseal \
      $(cat $JAIL/root.unseal.json \
        | jq -r ".unseal_keys_b64[$i]" \
        | base64 --decode \
        | gpg -dq \
      )
    i=$(( i + 1 ))
done
```

### greenfield: create root token, initialize and unseal database

- all green field vault instances require human intervention
  - create root pgp key
  - create root token and unseal database
  - create admin token
  - store root token next to your grenade launchers
- auto unseal: all have costs
  - alicloud kms
  - aws kms
  - azure key vault
  - gcp cloud kms
  - oci kms
  - hsm pkcs11
- auto unseal: free
  - vault transit
- preferred automation: auto unseal with seal transit
  - couldnt get working: limited capacity to troubleshoot
- preferred automation: auto unseal with aws kms
  - [cant afford it](https://aws.amazon.com/kms/pricing/) see alternative automation 1

```sh
######################### cd /nirv/core: create a root gpg key
# create root and admin gpg keys if they dont exist in jail
gpg --gen-key # repeat for for each entity (root, admin, ...) being assigned a gpg key

# base64 encode the `gpg: key` value of each gpg-key to $JAIL/_NAME_.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/root.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/admin_vault.asc

######################### start the vault with a green state
# stop any running vault containers
docker compose down

# remove previous vault data {e.g. raft,vault}.db
sudo rm -rf $VAULT_INSTANCE_DIR/data/*

# forcefully sync vault dev configs into the vault instance config dir
rsync -a --delete ../configs/vault/ $VAULT_INSTANCE_DIR/config

# finally: reset the vault server
./script.reset.compose.sh core_vault


######################### alternative method 1: CLI init & unseal
# confirm `Vault is not initialized`
vault operator init -status

# inititialize vault & distribute each unseal_keys_b64 to the appropriate people
## -n=key-shares (must match the number of pgp keys you created earlier)
## -t=key-threshold (# of key shares required to unseal)
vault operator init \
  -format="json" \
  -n=2 \
  -t=2 \
  -root-token-pgp-key="$JAIL/root.asc" \
  -pgp-keys="$JAIL/root.asc,$JAIL/admin_vault.asc" > $JAIL/root.unseal.json

# verify `Vault is initialized
vault operator init -status

# unseal vault: will require you to enter password set on the root pgp key
## see setup section for unseal cmd

# delete your shell history:
history -c

```

### greenfield: use root token to create vault admin token and policy

```sh
# verify you can access vault with root token
export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)
./script.vault.sh list secret-engines

# create policy for vault administrator
./script.vault.sh create poly $VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/policy_admin_vault.hcl
# create token for vault administrator
./script.vault.sh create token child $VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/token_admin_vault.json > $JAIL/admin_vault.json

# set VAULT_TOKEN to the new admin and verify access
## open $JAIL/admin_vault.json and ensure its valid json
export VAULT_TOKEN=$(cat $JAIL/admin_vault.json | jq -r '.auth.client_token')

# verify admin can login via UI via the browser
# verify token configuration
./script.vault.sh get token self
# verify admin access
./script.vault.sh list secret-engines

# restart the vault server and verify the admin token can unseal it
## restart vault (then rerun unseal cmd in previous step)
./script.refresh.compose.sh core_vault
```

### greenfield: use admin token to sync policies

- requires env vars set in previous step

```sh
# ensure vault_token points to the admin token
# then run unseal cmd in previous step
export VAULT_TOKEN=$(cat $JAIL/admin_vault.json | jq -r '.auth.client_token')

# forcefully sync vault dev configs into vault app
rsync -a --delete ../configs/vault/ $VAULT_INSTANCE_DIR/config

# initial policies
for file_starts_with_policy_ in $VAULT_INSTANCE_DIR/config/001-000-policy-init/*; do
  case $file_starts_with_policy_ in
  *"/policy_"*) ./script.vault.sh create poly $file_starts_with_policy_ ;;
  esac
done
```

### greenfield: configure secret engines

````sh
# the vault addr & path to chroot jail is required in all steps

# verify you can access vault with root token
export VAULT_TOKEN=$(cat $JAIL/root.asc.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)
./script.vault.sh list secret-engines
## scripts

- [scripting architecture & guidance](../scripts/README.md)

### script.vault.sh

- actively used for interacting with a secured vault server behind a secured proxy
- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.vault.sh)

```sh

####################### requirements
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/

####################### interface
export VAULT_ADDR=$VAULT_ADDR
export VAULT_TOKEN=$VAULT_TOKEN
# run `unset NIRV_SCRIPT_DEBUG && history -c` when debugging complete
export NIRV_SCRIPT_DEBUG=1

####################### configuring postgres dynamic creds
####################### creating new database engine
- disable verify connection
- create connection to postgres



####################### usage
./script.vault.sh poop poop poop

# enable a secret engine e.g. kv-v2
enable secret secretEngineType

# enable approle engine e.g. approle
enable approle approleType

# list all approles
list approles

# list enabled secrets engines
list secret-engines

# list provisioned keys for a postgres role
list postgres leases dbRoleName

# create a secret-id for roleName
create approle-secret roleName

# upsert approle appRoleName with a list of attached policies
create approle appRoleName pol1,polX

# create kv2 secret(s) at secretPath
# dont prepend `secret/` to secretPath
# e.g. create secret kv2 poo/in/ur/eye '{"a": "b", "c": "d"}'
create secret kv2 secretPath jsonString

# get dynamic postgres creds for database role dbRoleName
get postgres creds dbRoleName

# get the secret (kv-v2) at the given path, e.g. foo
# dont prepend `secret/` to path
get secret secretPath

# get the status (sys/healthb) of the vault server
get status

# get vault credentials for an approle
get creds roleId secretId

# get all properties associated with an approle
get approle info appRoleName

# get the approle role_id for roleName
get approle id appRoleName

# get the openapi spec for some path
help some/path/

````
