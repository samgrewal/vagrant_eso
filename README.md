# vagrant_eso

This repo contains a Vagrant file + dependencies that provisions a Centos 7 guest VM as a dockerized HTTP reverse proxy to a user-specified resource.

The Vagrantfile forwards ports `:22` and `:80` from the guest OS and binds them to `:2222` and `:8085` on the loopback interface of the host OS, so they are not publicly reachable. 

## TL;DR

```bash
$ git clone git@github.com:samgrewal/vagrant_eso.git
$ cd vagrant_eso
$ vagrant up
$ curl -sI http://localhost:8085
```

##  Prerequisites

This project has been validated on a Windows 10 Host with Vagrant 2.2.10 and Virtualbox 6.1. 

The following tutorials are excellent for installing GitBash, Vagrant, and Varnish on Windows 10. See [1] [2]

Of note, you will likely have to disable a number of BIOS features such as SecureBoot and Windows features that use VT-x to get Virtualbox 6.1 to work. Review these if you see a kernel panic on VM boot. See [4]

If the VM appears to partially boot, without ssh, and you see a "guest additions not installed" error in the console output, explore installing guest additions. See [2]

## File Structure

```
└───provision
    │   run_playbook.yml
    │
    └───roles
        ├───nginx-container
        │   ├───files
        │   │       default.conf.j2
        │   │       Dockerfile
        │   │
        │   ├───tasks
        │   │       main.yml
        │   │
        │   └───vars
        │           main.yml
        │
        └───security-base
            └───tasks
                    main.yml
```

## Roles

The `ansible_local` provisioner will apply 2 roles to the VM, `security-base` which asserts basic security hardening of the OS and `nginx-container` which provisions an nginx container on the guest OS as a reverse proxy. 

###  security-base
This role enforces the following configuration: 
- Blanket package update
- Disable the default unnecessary network applications `rpcbind` and `postfix`
- Disable root login and password authentication
- Configure a host-based firewall to only allow incoming traffic for ssh and http

### nginx-container

This role enforces the following configuration:
- Uses Jinja2 templating to create an nginx configuration file for a HTTP reverse proxy based on the `proxy_backend` and `proxy_scheme` role variables
- Installs and runs docker
	- *NOTE:* Docker commands are run with root privileges here because the normal process of adding the `ansible_user_id` to a docker shared group could not be implemented in an idempotent way trivially. There appears to be an open issue with the ansible `reset_connection` meta task on v2.9. Which would normally facilitate re-logging on the ansible user after group changes. See [3]
- Builds a docker image called `nginx:eso_proxy` from `nginx:latest` using the pre-templated configuration 
- Deploys a running instance of `nginx:eso_proxy`
	- *NOTE:* This is currently implemented with shell commands because the modern ansible modules `docker_image` and `docker_container` require the `docker` python module installed via pip. pip is not installed by default, so I did not want to risk introducing dependency issues in the limited scope of the project. 
- Once successful, you will be able to use the reverse proxy via a browser on your host machine through `http://localhost:8085`. 
	- *NOTE:* Top-level requests on the web page that change the domain will cause you to bypass the proxy.

## Next Steps

- Configure the nginx reverse proxy to support SSL/TLS termination. This may take place 1 of 2 ways:
	- Use a valid public TLD (.com, .net) and request a certificate from a public CA like Digicert or Letsencrypt. 
	- Use any TLD (.internal, .private) and generate a self-signed certificate. 
	- Web clients on the host OS will need to resolve the domain in either case through use of the hosts file or DNS.
- Explore getting the `nginx:eso_proxy` container to run without root privileges. 
- Explore an efficient way of installing the `docker` python module which will enable the modern `docker_image` and `docker_container` ansible modules for a more idempotent deployment.

## References
- [1] https://sloopstash.com/blog/how-to-build-vm-on-windows-10-using-virtualbox-vagrant-git-bash.html
- [2] https://medium.com/@botdotcom/installing-virtualbox-and-vagrant-on-windows-10-2e5cbc6bd6ad
- [3] https://github.com/ansible/ansible/issues/66414
- [4] https://forums.virtualbox.org/viewtopic.php?f=1&t=62339#:~:text=Re%3A%20I%20have%20a%2064bit,can't%20install%2064bit%20guests&text=If%20you%20want%20to%20be,changing%20the%20Hyper%2Dv%20setting.

