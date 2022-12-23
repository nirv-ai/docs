# nirvai docs

- documentation for VAULT @ nirvai
- [vault scripts](https://github.com/nirv-ai/scripts)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## VAULT

- vault is an opensource database for sensitive data

### the philosophy at NIRVai

> if you cant commit it, vault it

### Why Vault

- NIRVai is an opensource-first platform, thus our security measures will be more extreme than others

### if your application has network access

- remove all your `.env` and other configuration files from gitignore
- move all your sensitive data into vault
- refactor your application to be resiliant despite unmet requirements (where are my creds?)
- ensure your application encrypts in transit
  - if persisting to disk, dont, fkn vault it
- commit your remaining data & configuration to git (which should contain no PII/etc)

### if your application is `air-gapped`

- continue with 12-factor

### How do I know if this datum is sensitive?

- it is
