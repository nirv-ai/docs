# nomad TLS

- todo: move this into cfssl readme
- ^ it can be used with both nomad & consul

- [hashicorp docs](https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls)

## setup TLS for mad.nirv.ai:4646

```sh
############################## initial setup of nomad TLS + cloudflare CFSSL
# install clouflare cfssl
sudo apt install golang-cfssl

# create a directory to store the keys
mkdir tls && cd tls

# generate priv key and cert
## nomad-ca-key.pem: used to sign certs for nomad nodes and must be kept private
## nomad-ca.pem contains public key for validating nomad certs
##  ^ must be distributed to every node requiring access
cfssl print-defaults csr | cfssl gencert -initca - | cfssljson -bare nomad-ca

# create a cfssl.json (see https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls)
# generate certs for the nomad server
echo '{}' | cfssl gencert -ca=nomad-ca.pem -ca-key=nomad-ca-key.pem -config=cfssl.json \
    -hostname="server.global.nomad,localhost,127.0.0.1,mad.nirv.ai,dev.nirv.ai" - | cfssljson -bare server

# generate certs for the client server
echo '{}' | cfssl gencert -ca=nomad-ca.pem -ca-key=nomad-ca-key.pem -config=cfssl.json \
    -hostname="client.global.nomad,localhost,127.0.0.1,mad.nirv.ai,dev.nirv.ai" - | cfssljson -bare client

# generate certs for cli communication
echo '{}' | cfssl gencert -ca=nomad-ca.pem -ca-key=nomad-ca-key.pem -profile=client \
    - | cfssljson -bare cli


# now go to wherever you run nomad commands and symlink the TLS dir
# cd ~/git/private/nirv/core/apps/nirvai-web-nomad/dev
ln -s /path/to/where/you/keep/secrets/tls .

# generate a gossipkey and add to the server configuration
script.nmd.sh create gossipkey
# nomad should already be configured to use the tls directory
# for both client & server

# review what each file is intended for: https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls
# Each Nomad node requires a (POOP-key.pem) and certificate (POOP.pem)
# file for its region and role.
# AND the CA's public certificate (nomad-ca.pem)

```

## nomad TLS architecture

- nomad certs do not use FQDNs for Common Name identifiers (e.g. like a website)
- nomad authenticates certs for nodes by signing them with their region and role
  - client.global.nomad: for a client node in the global region
  - server.us-west.nomad: for a server node in the us-west region
- nomad uses the same port for http and https
  - your server should either use tls or not
