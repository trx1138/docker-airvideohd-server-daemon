![Air Video Server HD Logo](icon/airvideohd-icon.png)

This is an Unofficial Docker container for InMethod's Air Video Server HD based on freely available Linux binaries at [http://www.inmethod.com](http://www.inmethod.com)

# Air Video HD Server Docker Container

[![](https://images.microbadger.com/badges/version/madcatsu/airvideohd-server-daemon.svg)](https://hub/docker.com/r/madcatsu/airvideohd-server-daemon) [![](https://images.microbadger.com/badges/image/madcatsu/airvideohd-server-daemon.svg)](https://microbadger.com/images/madcatsu/airvideohd-server-daemon) [![](https://app.buddy.works/madcatsu/docker-airvideohd-server-daemon/pipelines/pipeline/65951/badge.svg?token=a2f16774471e9c2ba3a444bde08f819c67c7e082f0505a4dbf52b3d56c53e9a9 "buddy pipeline")](https://app.buddy.works/madcatsu/docker-airvideohd-server-daemon/pipelines/pipeline/65951) [![](https://app.buddy.works/madcatsu/docker-airvideohd-server-daemon/pipelines/pipeline/65952/badge.svg?token=a2f16774471e9c2ba3a444bde08f819c67c7e082f0505a4dbf52b3d56c53e9a9 "buddy pipeline")](https://app.buddy.works/madcatsu/docker-airvideohd-server-daemon/pipelines/pipeline/65952) [![](https://img.shields.io/docker/pulls/madcatsu/airvideohd-server-daemon.svg)](https://hub/docker.com/r/madcatsu/airvideohd-server-daemon)

#### Github Repository - [https://github.com/madcatsu/docker-airvideohd-server-daemon](https://github.com/madcatsu/docker-airvideohd-server-daemon)

---

Air Video HD allows you to watch videos streamed instantly from your computer on your iPhone, iPad, iPod Touch or Apple TV. The Air Video iOS client is a purchasable universal application on the App Store.


## Usage

### Normal Container Operation

First things first, we need to prep network and storage configuration for the container before bringing the container up. Air Video HD is capable of adjusting transcoding bit rates for mobile devices and over the Internet streaming of video as well as serving video to clients on local area networks (WiFi and Ethernet), the next steps outlined describe the pre-requisites for local and Internet based streaming.

1. On the Docker Host, create folders for the configuration file, logs, and transcoding scratch area. Also make a note of the top level folder in which video files will be accessible from.

  >**Transcoding:** It's worthwhile noting that if you expect ideal performance for transcoding of videos from one format to those natively supported on your iOS device, the transcoding folder should be placed on storage that is suitable for this purpose. Flash or RAM disk are ideal but not necessary, so expect your mileage to vary depending on the source video quality.

2. Prep your network. You will need to provide a NAT/firewall rule on your router/perimeter network device to allow incoming TCP/UDP traffic to a single port on the container. This is an exercise for the reader to determine how best to implement.

3. For Internet accessible streaming of media, only port 45633 is required. If your router/firewall supports NAT forwarding of both TCP and UDP packets, configure both for this port, otherwise TCP will suffice. The NAT/firewall rule should direct traffic to the IP address of the Docker host, not the internal IP dynamically given to the container. UDP port 5353 does not require a NAT/firewall rule as it is used only for local service discovery on local networks.

4. On the host running Docker, from the command line enter:

```shell
docker run -d --name=<container name> --restart unless-stopped \
-v <config path>:/config -v <log path>:/logs \
-v <media path>:/media -v <transcode path>:/transcode \
-v /etc/localtime:/etc/localtime:ro \
-e AVHSPASS="password" \
-p 5353:5353/udp \
-p 45633:45633 \
-p 45633:45633/udp \
madcatsu/airvideohd-server-daemon:latest --create-user abc:<UID>:<PID>
```

Parameters you will need to change are surrounded by `<>` marks. A brief description of each of these and their purpose follows:

### Container Runtime Options

* `-d` - starts the container in "daemon" mode, in other words, it doesn't start and drop you into an interactive shell or stdin by default. The container processes will run in the background as services/daemons.
* `--name=<container name>` - give the container a useful name you can recognise it by later on. If you plan on running multiple instances of this container with different data, config and log locations and a different TCP port, it's recommended to add a numeric identifier to the container name that increments for each container deployed
* `--restart unless-stopped` - ensures the container will restart automatically on host restart or a crash of a container process

### Container Bind Mounts

* `<config path>` - the is a folder on the Docker host file system that contains the configuration file
* `<log path>` - this is a folder on the Docker host file system that logs are output to
* `<media path>` - this will be the base folder from which any media is shared and served by the container, feel free to create multiple folders underneath this top level folder and create share configurations as required for each sub-folder
* `<transcode path>` - this folder is entirely optional but recommended as it will keep the size of the containers overlay or aufs filesystem small and improve performance of media transcoding. Ideally this would be a folder located on an SSD disk/volume
* `/etc/localtime:/etc/localtime:ro` Will capture the local host system time for log output. Ensure you use the 'read only' option as denoted in the example. _If you prefer UTC output, you can skip this bind mount._

### Container Environment Variables

* `-e AVHSUPASS="<password>"` - you can set a simple password here and the container will secure a basic media share configuration with this password. To setup multiple users and passwords, refer to the more detailed configuration examples below.

### Container Port Mappings

* `-p 5353:5353/udp` - this port is used my mDNS for Zeroconf discovery of the Air Video HD server from devices that support this protocol
* `-p 45633:45633` and `-p 45633:45633/udp` - these TCP and UDP ports are used by clients connecting to the Air Video HD Server container for streaming media

### Container Entrypoint and User mapping

* `--create-user abc:<UID>:<GID>` - The container leverages the [Chaperone](http://garywiz.github.io/chaperone/index.html) supervisor and init system which allows users to specify a user and group from the Docker host machine to run the in-container processes and access any bind mounts on the host without messing up permissions which can easily occur when processes in a container run as "root".

  The container avoids this issue by allowing users to specify an existing Docker host user account and group with the `UID` and `GID` environment variables.

  > To lookup the User and Group ID of the Docker host user account, enter the following command in the CLI on the Docker host as below:

  > + `id -u <insert your username here>`
  > + `id -g <insert your username here>`

### Air Video HD Server Configuration
After the container has run for the first time, a configuration file `Server.properties` is created in the `/config` bind mounted folder on the Docker host. Using a text editor of choice, the default settings can be modified to allow multiple authenticated users to connect to Air Video Server HD with password security and if desired, folder isolation.

Once changes to the file are saved, the container can be restarted to commit these to the running container configuration. Alternatively, you can call `curl -q http://localhost:45601/reloadSettings` via `docker exec` against the running container. See the notes below under "Info" for the syntax of the command.

Also, note that during container first run and initialization, the Server properties are updated to set the name of the Air Video Server to the container name.

**Caution:** _This properties file could be opened by anyone unless you set the appropriate umask on the file so that only the account that the container runs the service as has access. Passwords here are stored in clear text, you have been warned._

---

#### Example Configuration - Single User with no password

Edit Server.properties and replace the Sharing section with the following entries:

```
#
# Sharing settings
#

# First shared folder
sharedFolders1.displayName = Movies
sharedFolders1.path = /media/Movies

# Second shared folder
sharedFolders2.displayName = TV Shows
sharedFolders2.path = /multimedia/TV\ Shows

# multiuser mode (true/false)
multiUserMode = false

# single user mode password
singleUserPassword =
```

---

#### Example Configuration - Single User with password

Edit Server.properties and replace the Sharing section with the following entries:

```
#
# Sharing settings
#

# First shared folder
sharedFolders1.displayName = Movies
sharedFolders1.path = /media/Movies

# Second shared folder
sharedFolders2.displayName = TV Shows
sharedFolders2.path = /multimedia/TV\ Shows

# multiuser mode (true/false)
multiUserMode = false

# single user mode password
singleUserPassword = 1q2w3e4r <---- change this!
```

---

#### Example Configuration - Multiple Users with password

Edit the Server.properties and replace the Sharing section with the following entries:

```
#
# Sharing settings
#

# First shared folder
sharedFolders1.displayName = Movies
sharedFolders1.path = /media/Movies

# Second shared folder
sharedFolders2.displayName = TV Shows
sharedFolders2.path = /multimedia/TV\ Shows

# Third shared folder
sharedFolders3.displayName = Training
sharedFolders3.path = /multimedia/Training

# multiuser mode (true/false)
multiUserMode = true

# First user account (can access all folders)
userAccounts1.accessAllFolders = true
# userAccounts1.allowedFolders =
userAccounts1.userName = tom
userAccounts1.password = cat <---- change this!

# Second user account (can access selected folders)
userAccounts2.accessAllFolders = false
userAccounts2.allowedFolders1 = 1
# userAccounts2.allowedFolders2 = 2
userAccounts2.userName = richard
userAccounts2.password = third <---- change this!

# Third user account (can access selected folders)
userAccounts3.accessAllFolders = false
userAccounts3.allowedFolders1 = 2
userAccounts3.allowedFolders2 = 3
userAccounts3.userName = harry
userAccounts3.password = potter <---- change this!
```

### Connecting to AirVideoServer HD

Once the container is running, connecting to it from an iOS client is relatively easy, there are two methods to connect and register a new server in the iOS application.

1. Once the container has been run for the first time, you can register a client using a unique PIN assigned to the container. The PIN can be found in the `/config/.AirVideoServerHD/ApplicationData/Application.prefs` file stored in the bind mounted config directory that the container uses.

2. Alternatively, you can connect to the AirVideoServerHD server by specifying the hostname address or IP address of the host running the Docker container. Assuming you have exposed port 45633 as per the previous directions, the iOS client should discover and connect to the server.  

## Info

+ Shell access whilst the container is running: `docker exec -it <container name / ID> /bin/bash`
+ To monitor logs of the container in real time: `docker logs -f <container name / ID>`
+ To reload Air Video HD Server and reload settings without restarting the container: `docker exec -it <container name / ID> curl -q http://localhost:45601/reloadSettings`

## Known Issues

* ~~On first startup of the container, initially the shared folder(s) are not visible. To resolve this issue, simply restart the container after which the Sharing.prefs file is created correctly.~~

## To Do

- [ ] Review and consider implementing Media Sharing Settings via container environment variables

## Versions

+ **2017/09/20:**
  * Container initial release - Replaces legacy version on Docker Hub
