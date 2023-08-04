---
layout: default
title: The Eye Of Satoshi
nav_order: 20
parent: Bitcoin
---
<!-- markdownlint-disable MD014 MD022 MD025 MD033 MD040 -->

# The Eye Of Satoshi
{: .no_toc }

We set up [teos](https://github.com/talaia-labs/rust-teos){:target="_blank"} to serve as a watchtower for CLN Lightning node.

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Watchtower

Lightning channels need to be monitored to prevent malicious behavior by your channel peers. 
If your RaspiBolt goes down for a longer period of time, for instance due to a hardware problem, a
node on the other side of one of your channels might try to close the channel with an earlier channel balance that is better for them.

The Eye Of Satoshi can monitor your channels for you. If they detect such bad behavior, they can react on your behalf, and send a punishing transaction to close this channel.

---

## Preparations

Make sure that you have [reduced the database cache of Bitcoin Core](bitcoin-client.md#reduce-dbcache-after-full-sync) after full sync.

### Install dependencies

* Install build tools needed to compile Teos from the source code

  ```sh
  $ sudo apt-get update
  $ sudo apt install cargo clang cmake rustfmt libssl-dev
  ```

## The Eye Of Satoshi

[The Eye of Satoshi](https://github.com/talaia-labs/rust-teos){:target="_blank"} is a Lightning watchtower compliant with [BOLT13](https://github.com/sr-gi/bolt13), written in Rust.
There are no binaries available, so we will compile the application ourselves.

### Build from source code

We get the latest release of the Teos source code, verify it, compile it to an executable binary and install it.

* Download the source code for the latest Electrs release.
  You can check the [release page](https://github.com/talaia-labs/rust-teos/releases){:target="_blank"} to see if a newer release is available.
  Other releases might not have been properly tested with the rest of the RaspiBolt configuration, though.

  ```sh
  $ VERSION="0.2.0"
  $ mkdir /home/admin/rust
  $ cd /home/admin/rust
  $ git clone --branch v$VERSION git clone https://github.com/talaia-labs/rust-teos.git
  $ cd rust-teos
  ```

* To avoid using bad source code, verify that the release has been properly signed by the main developer [Roman Zeyde](https://github.com/romanz){:target="_blank"}.

  ```sh
  $ curl https://romanzey.de/pgp.txt | gpg --import
  $ git verify-tag v$VERSION
  > gpg: Good signature from "Roman Zeyde <me@romanzey.de>" [unknown]
  > gpg: WARNING: This key is not certified with a trusted signature!
  > gpg:          There is no indication that the signature belongs to the owner.
  > Primary key fingerprint: 15C8 C357 4AE4 F1E2 5F3F  35C5 87CA E5FA 4691 7CBB
  ```

* Now compile the source code into an executable binary and install it.
  The compilation process can take up to one hour.

  ```sh
  $ cargo build --locked --release
  $ sudo install -m 0755 -o root -g root -t /usr/local/bin ./target/release/teos-cli
  $ sudo install -m 0755 -o root -g root -t /usr/local/bin ./target/release/teosd
  ```

### Configuration

* Create the "teos" service user, and make it a member of the "bitcoin" and "debian-tor" group

  ```sh
  $ sudo adduser --disabled-password --gecos "" teos
  $ sudo usermod -a -G bitcoin,debian-tor teos
  ```

* Create the Teos data directory

  ```sh
  $ sudo mkdir /data/teos
  $ sudo chown -R teos:teos /data/teos
  ```

* Switch to the "teos" user, make link to .teos dir and create the config file with the following content

  ```sh
  $ sudo su - teos
  $ ln -s /data/teos /home/teos/.teos
  $ nano nano /data/teos/teos.toml
  ```

  ```sh
  # RaspiBolt: teos configuration
  # /data/teos/teos.toml

  # API
  api_bind = "127.0.0.1"
  api_port = 9814
  tor_control_port = 9051
  onion_hidden_service_port = 9814
  tor_support = true
  
  # RPC
  rpc_bind = "127.0.0.1"
  rpc_port = 8814
  
  # bitcoind
  btc_network = "mainnet"
  btc_rpc_user = "CSW"
  btc_rpc_password = "NotSatoshi"
  btc_rpc_connect = "localhost"
  btc_rpc_port = 8332
  
  # Flags
  debug = false
  deps_debug = false
  overwrite_key = false
  
  # General
  subscription_slots = 10000
  subscription_duration = 4320
  expiry_delta = 6
  min_to_self_delay = 20
  polling_delta = 60
  
  # Internal API
  internal_api_bind = "127.0.0.1"
  internal_api_port = 50051
  ```

* Let's start Teos manually first to check if everything runs as expected.

  ```sh
  $ teosd
  ```

  ```sh
  2023-08-04T06:46:12.340Z INFO [teosd] Loading configuration from file
  2023-08-04T06:46:12.341Z INFO [teosd] tower_id: 032a4ff7dd37ae01de848c3029c91253e0f30ee9038dcbcd08d6338bbbcb11339a
  2023-08-04T06:46:12.459Z INFO [teosd] Last known block: 000000000000000000006592a6929cb1db3f2f614b7921e2f778e8b54a2711bd
  2023-08-04T06:46:24.318Z INFO [teosd] Bootstrapping from backed up data
  2023-08-04T06:46:24.438Z INFO [teosd] Bootstrap completed. Turning on interfaces
  2023-08-04T06:46:24.438Z INFO [teos::api::tor] Loading Tor secret key from disk
  2023-08-04T06:46:24.441Z INFO [teosd] Starting up Tor hidden service
  2023-08-04T06:46:24.443Z INFO [teos::api::tor] Onion service: devrzfhfi2xay3ur5fpiualkqq6sedkaykvwsaf4in5rprohsoxfqoad.onion:9814
  2023-08-04T06:46:24.443Z INFO [teosd] Tower ready
  ...
  ```

* Stop Teos with `Ctrl`-`C` and exit the "teos" user session.

  ```sh
  $ exit
  ```

### Autostart on boot

Electrs needs to start automatically on system boot.

* As user "admin", create the Electrs systemd unit and copy/paste the following configuration. Save and exit.

  ```sh
  $ sudo nano /etc/systemd/system/electrs.service
  ```

  ```sh
  # RaspiBolt: systemd unit for electrs
  # /etc/systemd/system/electrs.service

  [Unit]
  Description=Electrs daemon
  Wants=bitcoind.service
  After=bitcoind.service

  [Service]

  # Service execution
  ###################
  ExecStart=/usr/local/bin/electrs --conf /data/electrs/electrs.conf

  # Process management
  ####################
  Type=simple
  Restart=always
  TimeoutSec=120
  RestartSec=30
  KillMode=process

  # Directory creation and permissions
  ####################################
  User=electrs

  # /run/electrs
  RuntimeDirectory=electrs
  RuntimeDirectoryMode=0710

  # Hardening measures
  ####################
  # Provide a private /tmp and /var/tmp.
  PrivateTmp=true

  # Use a new /dev namespace only populated with API pseudo devices
  # such as /dev/null, /dev/zero and /dev/random.
  PrivateDevices=true

  # Deny the creation of writable and executable memory mappings.
  MemoryDenyWriteExecute=true

  [Install]
  WantedBy=multi-user.target
  ```

* Enable and start Electrs.

  ```sh
  $ sudo systemctl enable electrs
  $ sudo systemctl start electrs
  ```

* Check the systemd journal to see Electrs' log output.

  ```sh
  $ sudo journalctl -f -u electrs
  ```

  Electrs will now index the whole Bitcoin blockchain so that it can provide all necessary information to wallets.
  With this, the wallets you use no longer need to connect to any third-party server to communicate with the Bitcoin peer-to-peer network.

* Exit the log output with `Ctrl`-`C`

### Remote access over Tor (optional)

To use your Electrum server when you're on the go, you can easily create a Tor hidden service.
This way, you can connect the BitBoxApp or Electrum wallet also remotely, or even share the connection details with friends and family.
Note that the remote device needs to have Tor installed as well.

* Add the following three lines in the section for "location-hidden services" in the `torrc` file.

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```sh
  ############### This section is just for location-hidden services ###
  HiddenServiceDir /var/lib/tor/hidden_service_electrs/
  HiddenServiceVersion 3
  HiddenServicePort 50002 127.0.0.1:50002
  ```

* Reload Tor configuration and get your connection address.

  ```sh
  $ sudo systemctl reload tor
  $ sudo cat /var/lib/tor/hidden_service_electrs/hostname
  > abcdefg..............xyz.onion
  ```

---

💡 Electrs must first fully index the blockchain and compact its database before you can connect to it with your wallets.
This can take a few hours.
Only proceed with the [next section](desktop-wallet.md) once Electrs is ready.

* To check if Electrs is still indexing, you can follow the log output

  ```sh
  $ sudo journalctl -f -u electrs
  ```

---

## For the future: Electrs upgrade

Updating Electrs is straight-forward.
You can display the current version with the command below and check the Electrs [release page](https://github.com/romanz/electrs/releases){:target="_blank"} to see if a newer version is available.

🚨 **Check the release notes!**
Make sure to check the [release notes](https://github.com/romanz/electrs/blob/master/RELEASE-NOTES.md){:target="_blank"} first to understand if there have been any breaking changes or special upgrade procedures.

* Check current Electrs version

  ```sh
  $ electrs --version
  ```

* If a newer release is available, you can upgrade by executing the following commands with user "admin"

  ```sh
  $ cd /home/admin/rust/electrs

  # Clean and update the local source code and show the latest release tag (example: v0.9.14)
  $ git clean -xfd
  $ git fetch
  $ git tag | sort --version-sort | tail -n 1
  > v0.9.14
  
  # Set the VERSION variable as number from latest release tag and verify developer signature
  $ VERSION="0.9.14"
  $ git verify-tag v$VERSION
  > gpg: Good signature from "Roman Zeyde <me@romanzey.de>" [unknown]
  > [...]

  # Check out the release
  # Should you encounter an error about files that would be overwritten use the -f argument to force the checkout
  $ git checkout v$VERSION

  # Compile the source code
  $ cargo clean
  $ cargo build --locked --release

  # Back up the old version and update
  $ sudo cp /usr/local/bin/electrs /usr/local/bin/electrs-old
  $ sudo install -m 0755 -o root -g root -t /usr/local/bin ./target/release/electrs

  # Update the Electrs configuration if necessary (see release notes)
  $ nano /data/electrs/electrs.conf

  # Restart Electrs
  $ sudo systemctl restart electrs
  ```

<br /><br />

---

Next: [Desktop wallet >>](desktop-wallet.md)
