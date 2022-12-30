## scripts

- [scripting architecture & guidance](../scripts/README.md)

### script.nmd.sh

- [source code](https://github.com/nirv-ai/scripts/blob/develop/nomad)
- [access nomad UI for nirvai-core @ https://mad.nirv.ai:4646](https://mad.nirv.ai:4646/ui/jobs)

```sh
###################### USAGE
## prefix all cmds with script.nmd.sh

# check on the server
get status team
get status all
dockerlogs # following logs of all running containers

# creating stuff
create gossipkey
create job myJobName
get plan myJobName # provides indexNumber

# running stuff
run myJobName indexNumber

# restarting stuff
restart loc allocationId taskName # todo: https://developer.hashicorp.com/nomad/docs/commands/alloc/restart

# execing stuff
exec loc  allocationId cmd .... @ todo https://developer.hashicorp.com/nomad/docs/commands/alloc/exec

# checking on running/failing stuff
get status node # see client nodes and there ids
get status node nodeId # provding a clients nodeId is super helpful; also provides allocationId
get status loc allocationId # super helpful for checking on failed jobs, provides deployment id
get status dep deploymentId # super helpful
get logs jobName deploymentId

# stopping stuff
stop job myJobName
rm myJobName # this purges the job
gc # purge system

```
