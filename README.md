# How to install Rstudio server without root previlage

On HPC, Rstudio server can only be installed by admin. The admin may download *.rpm file from the rstudio website and run `rserver` as a service (daemon). This process typically uses temporary folders that other users cannot modify (/etc and /var). However, users can run rserver separately from the admin's one with own configurations, such as using different version of R, different version of rstudio-server (preview version which support python)

This work is only tested in our HPC system, with centos 7.5

## install Rstudio server in user's home space

Just download rpm file and unpack locally.

```bash
# make installation path
mkdir -p /home/users/kjyi/rstudio-server
cd /home/users/kjyi/rstudio-server

# download
# This is a preview version of rstudio server with some experimental functions
wget https://s3.amazonaws.com/rstudio-ide-build/server/centos7/x86_64/rstudio-server-rhel-1.4.1100-x86_64.rpm

# unpack
rpm2cpio rstudio-server-rhel-1.4.1100-x86_64.rpm | cpio -idmv
rm rstudio-server-rhel-1.4.1100-x86_64.rpm
```

## how to run rstudio-server without root previlage

One must set all temporary file destinations to where the user can write (e.g. $HOME)
One have to provide following options when running the `rserver` proc

* `--server-pid-file=$HOME/rstudio-server/var/run/rstudio-server.pid`
* `--server-data-dir=$HOME/rstudio-server/var/run/rstudio-server`

Additionally, in preview version 1.4, one must also specify sqlite database file into modifiable path. This can be done by providing proper database.conf

```bash
# This code will make an invisible temporary folder at users's home directory and then make database.conf
mkdir -p $HOME/.rstudio-server
cat <<EOF > $HOME/rstudio-server/database.conf
provider=sqlite
directory=$HOME/rstudio-server
EOF
```

* Then you have to provide `--database-config-file $HOME/.rstudio-server/database.conf` when running rserver
* The rserver will detect PATH (`which R`) and use it as the main kernel. One can specify R binary by providing `--config-file`

```bash
# example config file
cat <<EOF > ~/.rstudio-sever/rsession.conf
rsession-which-r=/home/users/kjyi/bin/miniconda3/bin/R
EOF
```

## Running rstudio server

```bash
~/rstudio-server/usr/lib/rstudio-server/bin/rserver \
    --server-user kjyi \
	--server-daemonize=0 \
	--server-pid-file=$HOME/rstudio-server/var/run/rstudio-server.pid \
	--server-data-dir=$HOME/rstudio-server/var/run/rstudio-server \
	--www-address=0.0.0.0 \
	--www-port=1234 \
	--auth-none=0 \
	--database-config-file $HOME/rstudio-server/database.conf \
	--config-file $HOME/rstudio-server/rsession.conf
```

## etc 1: adjust timeout session

Rstudio server suspend user's session to disk when no command is running more than 2 hrs.
When working with very large objects, it is very painful when it happens. 
One can adjust this by specifying extended timeout session at the config-file (`rsession.conf`).

`session-timeout-minutes=240`

for infinity, specify it 0. 

## etc 2: access to sandbox node

```bash
# Run in your local computer terminal (wsl for Windows users)
# For security issue, I wrote any IP addresses and port below.
TUNNEL_IP=142.123.123.123
SERVER_MAINNODE_IP=142.123.123.123
SERVER_SANDBOX_ADDRESS=anode16

ssh -L 11010:localhost:11010 $TUNNEL_IP -p 3020 -t \
  ssh -L 11010:localhost:11010 $SERVER_MAINNODE_IP -p 3020 -t \
    ssh -L 11010:localhost:11010 $SERVER_SANDBOX_ADDRESS

# In here, the linking port is 11010, so you can open the sandbox's port 11010 with your local 11010 port
# If you run rstudio server with 11010 port on the sanbox node, you can open it by your browser with `localhost:11010` until the ssh session is alive.
```
