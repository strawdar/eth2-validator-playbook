Ethereum Proof of Stake Validator Playbook (GOERLI/PRATER)
==============================================================

This playbook configures a target system to run an Ethereum concensus client node + PoS validator using Lighthouse as well as an execution
client node using Geth. It is currently setup for GOERLI/PRATER in preparation for The Merge.

I wrote this for my own use so that I can quickly re/configure a validator from scratch if need be. I'm happy to share it, and hopefully it can be of use to others in the community.

Most of the playbook is based on [Somer Esat's very helpful Lighthouse staking guide](https://medium.com/@SomerEsat/guide-to-staking-on-ethereum-2-0-ubuntu-medalla-lighthouse-c6f3c34597a8). Parts of [Sigma Prime's Lighthouse Book](https://lighthouse-book.sigmaprime.io/become-a-validator-source.html) also came in handy.


Test Setup
----------
Automation loves to break. So while an attempt was made to keep things somewhat general, nothing is guaranteed unless your setup is 110% exactly the same as mine 😉

- Target System
    - Intel NUC8i5BEH, 32GB RAM, 2TB NVMe SSD, wired gigabit ethernet
    - Fresh install of Ubuntu 22.04 LTS Desktop (x86_64)
    - A non-root admin user available (sudo access)
    - SSH already setup with public key auth
- Controller System
    - Ansible 2.13.2 running from Arch Linux


Caveats
-------
- As is this playbook is x86_64 only (sorry Raspberry Pi). It's probably fairly close to working on ARM assuming that rustup works, but some of the downloads are also currently x86_64 specific.
- As is this playbook will only work on Ubuntu (or Debian-based) distributions. The main Debian-specific parts are the use of `apt` and the modification of `/etc/login.defs`.


Usage
-----

### Simple

If the target system's admin user is named `ubuntu` and SSH is running on the default port usage is pretty mundane. No, that comma after the IP address is not a typo.

NOTE: Ansible will ask for a "become" password, which is the target system's admin password needed to use sudo.

```
$ ansible-playbook validator.yml -i <target system ip>, --ask-become-pass
```

### Custom

To specify a differently named admin user the `PLAYBOOK_USER` environment variable can be used. A different SSH port can also be specified after the IP address.

In this example our admin user is `satoshi`, our IP address is `192.168.13.37`, and our SSH port is `2222`. Again, the comma after the port number is not a typo.

```
$ PLAYBOOK_USER=satoshi ansible-playbook validator.yml -i 192.168.13.37:2222, --ask-become-pass
```

### Multi

You should be able to manage multiple systems with a playbook like this and an inventory file, but I haven't tested it.


Post-Playbook Stuff
-------------------
- The playbook will not setup your validator keys with Lighthouse. This should be done as the `lhvc` user, which can be accessed with the `login-lhvc.sh` convenience script. When importing validator keys the value of `--datadir` should be `/var/lib/lighthouse`.
- If you want to setup graffiti for a POAP, modify the command in the systemd file at `/etc/systemd/system/lighthouse-beacon-node.service`.
- None of the new systemd services are configured to start on boot. Services that people probably want to enable at boot:
    - `geth.service`
    - `lighthouse-beacon-node.service`
    - `lighthouse-validator-client.service`
    - `prometheus.service`
    - `node_exporter.service`
    - `grafana-server.service`
- Login to Grafana and change the default admin password! The default login is just `admin`/`admin`. Grafana runs on port 3000 with HTTPS.
- The Grafana install has nothing setup. Refer to the Grafana section in Somer Esat's guide to get an initial dashboard up and running.
- Lighthouse is installed globally just like in Somer Esat's guide, and consequently the binary will need to be manually copied over to `/usr/local/bin/` whenever a new version is built. The source code is placed at `~/src/lighthouse` in the admin user's home directory.


Changes from Somer Esat's Guide
-------------------------------
- Restrictive umasks are set to prevent global readable files from being created by mistake.
- None of the new systemd services are configured to start on boot.
- The Go Ethereum user/group name is `geth`.
- The Lighthouse beacon node user/group name is `lhbn`.
- The Lighthouse beacon node systemd service name is `lighthouse-beacon-node.service`.
- The Lighthouse validator client user/group name is `lhvc`.
- The Lighthouse validator client systemd service name is `lighthouse-validator-client.service`.
- Both the `lhbn` and `lhvc` users are members of a common `lighthouse` group.
- Prometheus and node_exporter run under a single user/group named `metrics`.
- A self-signed certificate is generated and Grafana is configured to run over HTTPS.
- A set of convenience scripts are installed for the admin user at `~/bin` for tailing logs and switching to other users.


Upgrading from Lighthouse 0.2.x
-------------------------------
Previous versions of this playbook installed Lighthouse version 0.2.x. Version 0.3 brought
breaking database changes and also changes to how the data directory is handled. Older installations can be
upgraded with this current playbook, but some manual intervention will be required afterwards.

- The data directories have moved from `/var/lib/lhbn` and `/var/lib/lhvc` to sub-directories under
`/var/lib/lighthouse`. This playbook will create the new data directory for you and leave the old locations intact.
- The beacon node will need to be resync'd from scratch. Because the data directory is now at a different location this
will happen automatically when the beacon node service is started.
- After the beacon node successfully synchronizes with the head state of the chain, the contents of `/var/lib/lhvc` need
to be copied into `/var/lib/lighthouse/validators`. The file paths inside `validator_definitions.yml` will also need to
be updated to reflect the new data directory path.
- Once you have verified that your beacon node and validator client are up and running again with the new version of
Lighthouse feel free to delete the `/var/lib/lhbn` and `/var/lib/lhvc` directories to reclaim disk space.


Casual Disclaimer
-----------------

This software is provided as is with no warranty. It might burn your house down.


Tips
----

If people feel so inclined 💌
- ETH: strawdar.eth

Please also consider tipping Somer Esat for writing the original guide 🦄
- https://twitter.com/SomerEsat
- https://medium.com/@SomerEsat
