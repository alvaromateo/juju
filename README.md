<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/.sphinx/_static/logos/juju-logo-dark.png?raw=true">
  <source media="(prefers-color-scheme: light)" srcset="docs/.sphinx/_static/logos/juju-logo.png?raw=true">
  <img alt="Juju logo next to the text Canonical Juju" src="docs/.sphinx/_static/logos/juju-logo.png?raw=true" width="30%">
</picture>

Juju is an open source application orchestration engine that enables any application operation (deployment, integration, lifecycle management) on any infrastructure (Kubernetes or otherwise) at any scale (development or production) in the same easy way (typically, one line of code), through special operators called ‘charms’.

[![juju](https://snapcraft.io/juju/badge.svg)](https://snapcraft.io/juju)
[![snap](https://github.com/juju/juju/actions/workflows/snap.yml/badge.svg)](https://github.com/juju/juju/actions/workflows/snap.yml)
[![build](https://github.com/juju/juju/actions/workflows/build.yml/badge.svg)](https://github.com/juju/juju/actions/workflows/build.yml)


## Give it a try!

Let's use Juju to deploy, configure, and integrate some Kubernetes charms:


### Set up

You will need a cloud and Juju. The quickest way is to use a Multipass VM launched with the `charm-dev` blueprint.

[Install Multipass](https://canonical.com/multipass/docs/install-multipass). On Linux:

```
sudo snap install multipass
```

Use Multipass to launch an Ubuntu VM with the `charm-dev` blueprint:

```
multipass launch --cpus 4 --memory 8G --disk 50G --name tutorial-vm charm-dev
```

Open a shell into the VM:

```
multipass shell tutorial-vm
```

Verify that you have Juju and two localhost clouds:

```
juju clouds
```

Bootstrap a Juju controller into the MicroK8s cloud:

```
juju bootstrap microk8s tutorial-controller
```

Add a workspace, or 'model':

```
juju add-model tutorial-model
```

For architectures different than the default amd64 (i.e. macos laptops with the newer M1 chips), 
set the following constraint to the model:

```
juju set-model-constraints arch=arm64
```

### Deploy, configure, and integrate a few things

#### amd64 architecture

Deploy Mattermost:

```
juju deploy mattermost-k8s
```
> See more: [Charmhub | `mattermost-k8s`](https://charmhub.io/mattermost-k8s)

Deploy PostgreSQL:

```
juju deploy postgresql-k8s --channel 14/stable --trust
```

> See more: [Charmhub | `postgresql-k8s`](https://charmhub.io/postgresql-k8s)

Enable security in your PostgreSQL deployment:

```
juju deploy tls-certificates-operator
juju config tls-certificates-operator generate-self-signed-certificates="true" ca-common-name="Test CA"
juju integrate postgresql-k8s tls-certificates-operator
```

Integrate Mattermost with PostgreSQL:

```
juju integrate mattermost-k8s postgresql-k8s:db
```

Watch your deployment come to life:

```
watch -n 1 -c juju status --color
```

Use the `--relations` flag to view more information about your integrations.
Use the `--storage` flag to view more information about your storages.

#### arm64 architecture

The process to deploy the application when using an arm64 machine is a bit different, given that 
the mattermost-k8s charm doesn't have a built image for arm64.

First of all you'll need modify and build the 
[charm-k8s-mattermost](https://launchpad.net/charm-k8s-mattermost) charm. 

```
git clone https://git.launchpad.net/charm-k8s-mattermost
cd charm-k8s-mattermost
```

Now, you'll modify the charmcraft.yaml file so that it contains all the necessary information for 
the build. Overwrite charmcraft.yaml with the following content:

<details>
<summary>charmcraft.yaml</summary>
<br>

```yaml
name: mattermost-k8s
title: Mattermost
summary: Mattermost is a flexible, open source messaging platform that enables secure team collaboration.
links:
  documentation: https://discourse.charmhub.io/t/mattermost-documentation-overview/3758
  contact: launchpad.net/~canonical-is-sre
description: |
  Mattermost is a flexible, open source messaging platform that enables secure team collaboration.
  https://mattermost.com

assumes:
  - juju >= 2.8.0
  - k8s-api

requires:
  db:
    interface: pgsql
    limit: 1
    
type: "charm"
base: ubuntu@20.04
build-base: ubuntu@20.04

platforms:
  amd64:
  arm64:

containers:
  mattermost:
    resource: mattermost-k8s-image

resources:
  mattermost-k8s-image:
    type: oci-image
    description: OCI image for mattermost-k8s
```

</details>

Delete the metadata.yaml file, as it's used by older versions of charmcraft and you just added 
all the information that it contained to charmcraft.yaml.

```
rm metadata.yml
```

Next, install docker, build the OCI image from the Dockerfile and push it to the microK8s registry.

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install latest version
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Build image and push it to registry
sudo docker build -t localhost:32000/mattermost .
```

### Test your deployment

When everything is in `active` or `idle` status, note the IP address and port of Mattermost and pass them to `curl`:

```
curl <IP address>:<port>/api/v4/system/ping
```

You should see the output below:

```
{"AndroidLatestVersion":"","AndroidMinVersion":"","IosLatestVersion":"","IosMinVersion":"","status":"OK"}
```
### Congratulations!

You now have a Kubernetes deployment consisting of a Mattermost backed by PosgreSQL with TLS-encrypted traffic!

### Clean up

Delete your Multipass VM:

```
multipass delete --purge tutorial-vm
```

[Uninstall Multipass](https://canonical.com/multipass/docs/install-multipass). On Linux:

```
snap remove multipass
```

## Next steps

- Read the [docs](https://canonical-juju.readthedocs-hosted.com).
- Read our [Code of conduct](https://ubuntu.com/community/code-of-conduct) and join our [chat](https://matrix.to/#/#charmhub-juju:ubuntu.com) and [forum](https://discourse.charmhub.io/) or [open an issue](https://github.com/juju/juju/issues).
- Read our [CONTRIBUTING guide](./CONTRIBUTING.md) and contribute!
