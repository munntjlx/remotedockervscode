# How to remote develop in ec2 instance with docker and VSCODE

This document describes how to setup and get working a fully functional remote development environment in docker and ec2 instances. This is particularly useful when remote 'testing' containers prior to fargate or other types of install. I can create a remote 'aws ec2  instance' and remotely copy 'local' files on a remote. I use this for testing IAM permissions for cross-account roles that you can't do on non ec2 instances.

## Requirements

1. fully working passwordless ssh sudo not required except for permission to run 'docker command'. This does require you to configure docker to allow your user to execute docker commands (essentiall root anyway) just add your user to the docker group
2. install the 'remote containers extension from microsoft'
3. autossh - to reliably forward connections to your docker remote daemon
4. special configuration file to ensure that docker starts docker daemon on localhost 
5. Amazon linux 2 (you could probably use another version, but I cover that here)
6. docker environment variables if you want to do things outside of vs code
7. vscode settings file to allow you to run docker remotely (essentially the set commnands to get docker working in your settings.json file)
8. A development container (or dockerfile at least) that you want to run on the remote. The system MUST have tar, and possibly other files in order to work with this. A python/perl container is used here as an example

## Configure remote host to have docker on it

Procedure:

Install amazon linux 2
```bash
amazon-linux-extras install docker
service docker start
usermod -a -G docker ec2-user
 mkdir -p /etc/systemd/system/docker.service.d/
 vi /etc/systemd/system/docker.service.d/startup_options.conf
```
This file should have the following contents:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H unix:// -H tcp://127.0.0.1:2376
```
Reconfigure systemd and docker

```bash
   systemctl stop docker
   systemctl daemon-reload
   systemctl start docker
   netstat -tunlp | grep -i docker
   systemctl enable docker
```

You will see the display below if everything is working

```html
tcp        0      0 127.0.0.1:2376          0.0.0.0:*               LISTEN      12791/dockerd  
```

Once this is done, you should be ready to portforward do the docker daemon on your remote system. I assume you already know how to use ssh and that you have properly setup passwordless ssh etc, and that you are using ec2-user.

## setup on mac

```bash
brew install autossh jq
```

If you are using bash, you will have to convert this abbreviation (fish format) to bashese alias The important bit is the autossh cli stuff:

```bash
abbr dockertunnel autossh -f -M 12345 -N ec2-user@yoursite.com -L 127.0.0.1:2222:127.0.0.1:2376
```

This command means that I would like to alias the dockertunnel command to create a control channel (for keeping the tunnel up on port 12345), send it to the background, connect as ec2-user@my.remote portforward local port 2222 to remote port 2376 and keep things open until I kill the tunnel manually. The magic of autossh is that it will keep the tunnel up in a stable fashion even if there is a remote timeout set on the remote side, since it constantly sends data across the control channel.

Once this is done, set your environment for testing thus (fish format)

```bash
set -gx DOCKER_HOST tcp://127.0.0.1:2222
```

This says to export my docker host to localhost on the port that we just created. Once you do this you are ready for testing. To test:

```bash
curl -X GET http://127.0.0.1:2222/images/json |jq
```

The output should look something like this (minus all the containers) thus:

```json
[
  {
    "Containers": -1,
    "Created": 1570844944,
    "Id": "sha256:67e50cb62aa6d5a51014beaf17a165857f52f28dd38fe9f45fce2b4a26c1d6ef",
    "Labels": {
      "description": "Changed to protect the innocent",
      "name": "Showng you that this works",
      "version": "5.4.0"
    ...
    "
```

Your display will be much shorter. If this works, it means you are ready for the configuration of vscode itself.

Micsoft has some somewhat complex directions, but here is the [link](https://code.visualstudio.com/docs/remote/containers) if you are intersted. We will be assuming that you have a running container that works. Use their directions if you need help at this point. 

Install the [Remote Development Extension pack](https://aka.ms/vscode-remote/download/extension)

Once this is done, you will need to add the following json entry to your settings.json in the appropriate vscode location for your system (you can also search in settings for the docker.host setting)

```json
"docker.host":"tcp://127.0.0.1:2222",
```

It is essential that you DONT add the

```json
"docker.tlsVerify": "0" // or "0"
```

File, since it will cause everything to fail. It would be nice if docker-mache worked with amazon linux 2, but alas it does not. I also don't want to manually setup all the certs whatnot. By tunneling through ssh, we have all the safety of tls with none of the headaches. You will also have to restart vscode to get this working.  By binding the docker daemon, and ensuring that our firewall on our ec2 instance (either firewalld or security group or better yet BOTH!) don't allow the docker daemon to be exposed, we are far safer than if we expose the daemon to world+dog.

## Daily use

At first its not quite apparant how to get things working.  I just created a container with a docker file (in fact if you just use your 'dockerfile') in our repo with the appropriate SET commands (remember to export your docker), it will build our LOCAL repo on the REMOTE docker machine. This is much less time consuming than having to put it in ECR every time we make a mistake. I can create a generic docker, run my debugging, language tools etc (you install these on the REMOTE WHILE THE CONTAINER IS RUNNING). In theory you could create a 'permanent' remote dev container, but I find this tiresome. You can also copy files just by dragging them from window 1 (remote) to window 2 (local).


