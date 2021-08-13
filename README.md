# rp-ha
Highly available Rocketpool deployment
=======

This is currently experimental and partially working.

Architecture:
- Rocketpool "stack" in docker swarm or k8s. This repo has been tested on docker swarm; k8s would require some adjustments, though likely nothing too drastic. One copy of each service runs; on a HA deployment with multiple managers and workers, ideally across multiple AZs, the failure of one node just means those rocketpool services get "spun up" on another node.
- Two or more execution clients - Geth or others that the node operator trusts. TLS-encrypted is assumed by haproxy in this example, I am using eth-docker for that.
- Two or more consensus clients - tested with Lighthouse and Teku, should work with Nimbus. TLS-encrypted is assumed by haproxy in this example, I am using eth-docker for that. Prysm would require changes because it doesn't do https://.
- Shared storage is a must. NFS works; or any other shared-storage idea for k8s.

Files in this repo.

- rocketpool.yml - the actual "stack", this would go into portainer or be deployed some other way. Things to adjust here:
  - The domain names for the haproxy load balancer in its service, under `aliases`. These have to match the ACLs in the haproxy.cfg file.
  - The domain name to be used for beacon node connection by the validator in its service, under `entrypoint` and then the line after `--beacon-nodes`
  - Logging. I've kept the awslogs stuff in as an example, adjust to where you want the logs
  - NFS mount. I've kept an AWS example in, adjust to where your shared storage is

The following files would live as "configs" in docker swarm or k8s and get referenced by the yml file that defines the stack

- prater-haproxy.cfg - the configuration file for haproxy
  - Adjust domain names in here, both for ACLs (what the rocketpool services connect to via the haproxy service) and backend (the servers things get load-balanced to)
  - Note: The server name has to be the FQDN of the server for HTTPS not WSS, the external check script relies on it. It otherwise gets an IP address, which won't work with how eth-docker does TLS via traefik.
    So if your server is available as `goerli-ec-a.example.com` for HTTPS and `goerli-ecws-a.example.com` for WSS, then the two lines for it would respectively read
    `server goerli-ec-a.example.com goerli-ec-a.example.com:443 check` for HTTPS and `server goerli-ec-a.example.com goerli-ecws-a.example.com:443 check` for WSS. The first part is the server name,
    which is arbitrary as far as haproxy is concerned, and the check script uses that to make some RPC calls; the second part is the server address, the location that haproxy will send actual traffic to.

- check-ccsync.sh - external check script for haproxy that verifies that the consensus client is synced and has at least N peers

- check-ecsync.sh - external check script for haproxy that verifies that the execution client is synced and has at least N peers

Preparing the shared storage

- Have an NFS mount, create dir rocketpool
- Follow steps at https://docs.rocketpool.net/guides/node/native.html#setting-up-the-binaries to grab config.yml and restart-validator.sh and create a settings.yml
- Only a couple changes needed in the config.yml, for the provider. We'll be using haproxy. The exact host names can be changed of course.
  - smartnode: validatorRestartCommand: becomes /.rocketpool/restart-validator.sh 
  - eth1: provider: becomes https://goerli-lb.yourdomain.net
  - eth1: wsProvider: becomes wss://goerliws-lb.yourdomain.net
  - eth2: provider: becomes https://prater-lb.yourdomain.net
- settings.yml cannot be empty. Create it with values from another rocketpool install, or be sure to run `rp service config` from inside the `node` container before running `rp node register`.
- restart-validator.sh gains this line at the very end, as its only executable line: `docker service update --force rocketpool_validator`

Interacting with the node

Open a `sh` on the container that the `rocketpool_node` service runs in. This has to be on a manager node in docker swarm because of that restart-validator.sh above. The rocketpool command is `rp` and
can be used to init a wallet, register a node, stake RPL, deposit minipools etc. Not working will be `rp logs` and anything that stops, starts or terminates services.

Currently not working

- `rp minipool status` - this should get fixed in RC6
- restart-validator.sh hasn't been tested, gated by `rp minipool` commands working


MIT Licensed
