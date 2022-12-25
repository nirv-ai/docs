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
######################### FYI
# in all examples: /chroot/jail = ../secrets/dev/apps/vault
# @see https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/

# havent had any success in curling hcl to vault http api, convert to json via this tool
# @see https://www.convertsimple.com/convert-hcl-to-json/

######################### cd /nirv/core: create a root gpg key
# the vault addr & path to chroot jail is required in all steps
export JAIL="../secrets/dev/apps/vault"
export VAULT_ADDR=https://dev.nirv.ai:8300

# create root and admin gpg keys if they dont exist in jail
gpg --gen-key # repeat for for each entity being assigned a gpg key

# base64 encode the `gpg: key` value of each gpg-key to $JAIL/_NAME_.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/root.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/admin_vault.asc

######################### start the vault with a green state
# stop any running vault containers
docker compose down

# remove previous vault data {e.g. raft,vault}.db
sudo rm -rf apps/nirvai-core-vault/src/data/*

# forcefully sync vault dev configs into vault app
rsync -a --delete ../configs/vault/ apps/nirvai-core-vault/src/config

# finally: reset the vault server
./script.reset.compose.sh core_vault


######################### alternative method 1: CLI init & unseal
# confirm `Vault is not initialized`
vault operator init -status

# inititialize vault
## -n=key-shares (must match the number of pgp keys you created earlier)
## -t=key-threshold (# of key shares required to unseal)
vault operator init \
  -format="json" \
  -n=2 \
  -t=2 \
  -root-token-pgp-key="$JAIL/root.asc" \
  -pgp-keys="$JAIL/root.asc,$JAIL/admin_vault.asc" > $JAIL/root.unseal.json
# you can now distribute each unseal_keys_b64 to the appropriate people

# unseal vault: will require you to enter password set on the root pgp key
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

# delete your shell history:
history -c


######################### alternative method 2: UI init and unseal
# unseal the database: open https://dev.nirv.ai:8200/
## select create a new raft cluster: atleast 5 key shares, at least 3 key threshold
## enter the root.asc file as text into both pgp key fields and download the keys
## dowload your bank keys, it should match the following
{
  "keys": ["SOME_KEY1", "SOME_KEY2", "SOME_KEY3", ...],
  "keys_base64": ["SOME_KEY1", "SOME_KEY2", "SOME_KEY3", ...],
  "root_token": "ROOT_TOKEN_GUARD_WITH_YOUR_LIFE"
}

```

### greenfield: use root token to create vault admin token and policy

```sh
# the vault addr & path to chroot jail is required in all steps
export JAIL="../secrets/dev/apps/vault"
export VAULT_ADDR=https://dev.nirv.ai:8300

# verify you can access vault with root token
export VAULT_TOKEN=$(cat $JAIL/root.asc.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)
./script.vault.sh list secret-engines

# create policy: vault admin
./script.vault.sh create poly apps/nirvai-core-vault/src/config/policy/admin_policy_vault.hcl
# create role: vault admin
./script.vault.sh create token child apps/nirvai-core-vault/src/config/role/admin_role_vault.json > $JAIL/vault_admin.json

# set VAULT_TOKEN  to the new admin and verify access
## open vault_admin.json and remove extra statements so its valid json
export VAULT_TOKEN=$(cat $JAIL/vault_admin.json | jq -r '.auth.client_token')

# verify token configuration
./script.vault.sh get token self
# verify admin access
./script.vault.sh list secret-engines

# restart the vault server and verify the admin token can unseal it
./scrift.refresh.compose.sh core_vault
vault operator unseal \
  $(cat $JAIL/root.asc.json \
    | jq -r '.unseal_keys_b64[0]' \
    | base64 --decode \
    | gpg -dq \
  )
```

### greenfield: use admin token to sync policies

```sh
# the vault addr & path to chroot jail is required in all steps
export JAIL="../secrets/dev/apps/vault"
export VAULT_ADDR=https://dev.nirv.ai:8300
export VAULT_TOKEN=$(cat $JAIL/vault_admin.json | jq -r '.auth.client_token')
vault operator unseal \
  $(cat $JAIL/root.asc.json \
    | jq -r '.unseal_keys_b64[0]' \
    | base64 --decode \
    | gpg -dq \
  )

# forcefully sync vault dev configs into vault app
rsync -a --delete ../configs/vault/ apps/nirvai-core-vault/src/config

# sync policies: copypasta the below in your terminal
for policy in apps/nirvai-core-vault/src/config/policy/*; do
  echo -e "creating policy with:\n$policy"
  ./script.vault.sh create poly $policy
done
```

### greenfield: configure secret engines

````sh
# the vault addr & path to chroot jail is required in all steps
export JAIL="../secrets/dev/apps/vault"
export VAULT_ADDR=https://dev.nirv.ai:8300

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
