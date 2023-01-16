# nirvai scripts

> CLI-as-a-Service

- guides for writing cross-platform, resiliant posix compliant shell scripts.
- nirvai scripts is a single interface to all CLIs supporting NIRVai platform components and packages
- [source code](https://github.com/nirv-ai/scripts)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## INSTALLATION

- wherever you've cloned this repo, add the path to your init file

```sh
# if `pwd` === home/poop/git/private/nirv/scripts
# update your ~/.bashrc
export PATH="/home/poop/git/private/nirv/scripts:$PATH"

# if you're an operator, also source the shell-init dir
for operator_script in /home/poop/git/private/nirv/scripts/shell-init/*.sh; do
	source $operator_script
done;

```

## best practices

### never use a low-level CLI directly, always create a wrapper

- if a cli is integral to the use of your service, create a wrapper for your future self and others
- platform wide upgrades: move to the latest version and update in a single place
- useful side effects: wouldnt it be great to run fmt, check, and validate on every nomad cmd?
  - we even run nomad format on vault and consul configuration files: hashicorp eslint baby
- standardized interfaces
- CLIs as a service

### always use #!/usr/bin/env [SHELL]

- consumers can then use whatever shell they prefer
- the default should be bash, if your on mac [upgrade it](https://itnext.io/upgrading-bash-on-macos-7138bd1066ba)

### be consistent

- a solid platform architecture reuses common patterns for common use cases
- reuse scripts/utils as much as possible to help enforce and build upon those standards

### use this boilerplate

- it enables your script to be in compliance with NIRVai best practices
- it provides everything in the utils dir, most importantly
  - platform env: env vars that provide a common interface across all scripts
  - platform debug: debug logic and debug echo
  - platform errors: trap error and echo error
  - platform get app: get the location of an arbitrary app or a dir within it
  - platform invalid request: inform the user we need better documentation, its not their fault
  - platform request sudo: never use sudo without telling consumers why

```sh
#!/usr/bin/env bash

set -euo pipefail

######################## SETUP
DOCS_URI='https://github.com/nirv-ai/docs/blob/main/your-script-dir/README.md'
SCRIPTS_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]%/}")" &>/dev/null && pwd)"

SCRIPTS_DIR_PARENT="$(dirname $SCRIPTS_DIR)"

# PLATFORM UTILS
for util in $SCRIPTS_DIR/utils/*.sh; do
  source $util
done

######################## INTERFACE
# group by increasing order of dependency
# add your vars here

# add vars that should be printed when NIRV_SCRIPT_DEBUG=1
declare -A EFFECTIVE_INTERFACE=(
  [DOCS_URI]=$DOCS_URI
  [SCRIPTS_DIR_PARENT]=$SCRIPTS_DIR_PARENT
)

######################## CREDIT CHECK
echo_debug_interface

# add aditional checks and balances below this line
# use standard http response codes
throw_missing_dir $SCRIPT_DIR_PARENT 500 "somethings wrong: cant find myself in filesystem"
throw_missing_program curl 404 '@see https://curl.se/download.html'
throw_missing_program jq 404 '@see https://stedolan.github.io/jq/'

######################## FNS
# keep your main script small and move utils in a subdir
# MY-SCRIPT-NAME UTILS
for util in $SCRIPTS_DIR/your-scripts-dir/utils/*.sh; do
  source $util
done

## add your script fns below this line

######################## EXECUTE
# expose your fns using this case statement
# KISS: keep-in scripts simple: unless your bpytop that shiz is sweeeet

cmd=${1:-''}
case $cmd in
  *) invalid_request
esac
```

## available scripts

- [cloudflare cfssl](../cfssl/README.md)
- [docker scripts](../docker/README.md)
- [haproxy](../haproxy/README.md)
- [hashicorp nomad](../nomad/README.md)
- [hashicorp vault](../vault/README.md)
- [letsencrypt](../letsencrypt/README.md)
