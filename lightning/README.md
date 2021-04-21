# Install LND and Tor

This manual documents how to build and run Lightning on Pi 4. The main challenge is 64-bit environment while Pi base operating system Rasbian is 32-bit. Fortunately Pi 4 hardware is 64-bit.

The reason for running LND in 64-bit mode is to avoid 32-bit related bugs such as https://github.com/lightningnetwork/lnd/issues/5196 - athough LND team is commited to supporting 32-bit mode, some of these bugs can take several days to fix. By the way, there are also some not well understood issues possbliy related to 32-bit mode https://github.com/mynodebtc/mynode/issues/512 so using 64-bit mode can be solid way to confirm if 32-bit mode is the problem.

Prerequisites:
 * Boot Pi in [64-bit mode](https://medium.com/for-linux-users/how-to-make-your-raspberry-pi-4-faster-with-a-64-bit-kernel-77028c47d653) 
 * Login in as unix account that has sudo


## Chroot for 64-bit environment

```
sudo adduser --disabled-password lightning
sudo apt install -y debootstrap schroot

cat << EOF | sudo tee /etc/schroot/chroot.d/bitcoin64
[lightning64]
description=builds that need 64-bit environment
type=directory
directory=/mnt/btrfs/lightning64
users=lightning
root-groups=root
profile=desktop
personality=linux
preserve-environment=true
EOF

sudo debootstrap --arch arm64 buster /mnt/btrfs/lightning64

sudo schroot -c lightning64 -- apt update
sudo schroot -c lightning64 -- apt upgrade -y
```

Make direcetories inside the data mount point:
```
sudo mkdir /mnt/btrfs/lightning64/mnt/btrfs/lightning
sudo mkdir /mnt/btrfs/lightning64/mnt/btrfs/lightning/src
sudo mkdir /mnt/btrfs/lightning64/mnt/btrfs/lightning/bin

sudo chown -R lightning /mnt/btrfs/lightning6464/mnt/btrfs/lightning
```



## Install needed packages
```
sudo schroot -c lightning64 -- apt install -y git build-essential libtool autotools-dev automake pkg-config libssl-dev libevent-dev bsdmainutils  libboost-dev libboost-system-dev libboost-filesystem-dev  libboost-chrono-dev libboost-program-options-dev  libboost-test-dev libboost-thread-dev  libminiupnpc-dev  libzmq3-dev libdb5.3++-dev
```




## Setup LND environment

Log-in as "lightning" user and setup symlinks


```
sudo su -l lightning
schroot -c lightning64

ln -s /mnt/btrfs/lightning/lnd-data ~/.lnd
ln -s /mnt/btrfs/lightning/gocode
ln -s /mnt/btrfs/lightning/lnd-e2e-testing
ln -s /mnt/btrfs/lightning/src
```



## Install Tor

```
sudo apt install tor
```

 * Minibank needs tor version **0.3.3.6** or above. Fortunaly Rasiban 10 already has that. On older distos [build tor from source](https://github.com/alevchuk/minibank/tree/first/tor#build-from-source). 
 * Minibank uses Tor for LND. Yet not for Bitcoin sync traffic because that seems to introduce delays.

1. Edit `/etc/tor/torrc` 
* Uncomment "ControlPort 9051"
2. Run 
```
sudo systemctl restart tor@default.service
```

Add lightning user to be part of the Tor group (e.g. it needs read permissions to /run/tor/control.authcookie )
```
sudo /usr/sbin/adduser lightning debian-tor
```


## Build Go
Follow instrutions under [alevchuk/minibank/go](https://github.com/alevchuk/minibank/blob/first/go/)


## Build LND

1. Install dependencies:
* Fun fact: build-essential contains `make`
```
sudo apt-get install build-essential
```

2. Log in as "lightning"
```
sudo su -l lightning
schroot -c lightning64
```

3. Download, build, and Install LND:
```
go get -d github.com/lightningnetwork/lnd
(cd $GOPATH/src/github.com/lightningnetwork/lnd && make clean && make && make install)
```