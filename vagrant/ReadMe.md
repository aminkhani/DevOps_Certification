# Vagrant + Packer on Ubuntu/Debian

Set up [Vagrant](https://developer.hashicorp.com/vagrant) and [Packer](https://developer.hashicorp.com/packer) on Ubuntu/Debian, use ready-made [Bento](https://github.com/chef/bento) boxes, and build your own boxes when you need to.

## Table of contents

- [Vagrant + Packer on Ubuntu/Debian](#vagrant--packer-on-ubuntudebian)
  - [Table of contents](#table-of-contents)
  - [Requirements](#requirements)
  - [1. Install Vagrant and Packer](#1-install-vagrant-and-packer)
    - [Add the HashiCorp GPG key](#add-the-hashicorp-gpg-key)
    - [Add the official HashiCorp repository](#add-the-official-hashicorp-repository)
    - [Install](#install)
    - [Verify](#verify)
  - [2. Use public Bento boxes](#2-use-public-bento-boxes)
  - [3. Use a box in a Vagrantfile](#3-use-a-box-in-a-vagrantfile)
  - [4. Build your own boxes](#4-build-your-own-boxes)
    - [Requirements](#requirements-1)
    - [Clone Bento](#clone-bento)
    - [Build a VirtualBox box](#build-a-virtualbox-box)
    - [Add your built box to Vagrant](#add-your-built-box-to-vagrant)
  - [Everyday commands](#everyday-commands)
  - [Troubleshooting](#troubleshooting)
  - [Security notes](#security-notes)

## Requirements

- Ubuntu or Debian host (64-bit)
- [VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads) installed
- `curl`, `gnupg`, `lsb-release`, `git`

```bash
sudo apt-get update
sudo apt-get install -y curl gnupg lsb-release git
```

> **Version note:** keep VirtualBox and Vagrant reasonably close in age.
> Bento boxes ship with newer Guest Additions, and a much older VirtualBox on
> the host can cause warnings or odd behavior. Check the Vagrant release notes
> for the VirtualBox versions your Vagrant release supports.

## 1. Install Vagrant and Packer

### Add the HashiCorp GPG key

`apt-key` is deprecated, so store the key in a dedicated keyring instead:

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

### Add the official HashiCorp repository

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

> On Debian derivatives where `lsb_release -cs` returns a codename HashiCorp
> doesn't publish, replace it with the matching Debian/Ubuntu codename.

### Install

Packer is a HashiCorp tool that automatically builds machine images from a template. Instead of installing an OS by hand, configuring it, and saving the result, you describe the image in code and Packer produces it the same way every time.

```bash
sudo apt-get update
sudo apt-get install -y vagrant packer
```

### Verify

```bash
vagrant --version
packer --version
VBoxManage --version
```

## 2. Use public Bento boxes

Bento publishes minimal, VirtualBox-ready boxes on
[Vagrant Cloud](https://portal.cloud.hashicorp.com/vagrant/discover/bento).

```bash
vagrant box add --provider virtualbox bento/ubuntu-24.04
vagrant box add --provider virtualbox bento/ubuntu-22.04
vagrant box add --provider virtualbox bento/debian-12
```

Manage downloaded boxes:

```bash
vagrant box list             # list local boxes
vagrant box outdated         # check for newer versions
vagrant box update           # update the box for the current project
vagrant box remove bento/ubuntu-22.04
vagrant box prune            # remove old versions
```

## 3. Use a box in a Vagrantfile

Create a project and generate a Vagrantfile:

```bash
mkdir my-vm && cd my-vm
vagrant init bento/ubuntu-24.04
```

Or write a minimal one yourself:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus   = 2
  end
end
```

A Debian example:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/debian-12"
end
```

Start it and connect:

```bash
vagrant up
vagrant ssh
```

## 4. Build your own boxes

Only needed if you want a custom image. Otherwise use the public boxes above.

### Requirements

Packer, Vagrant, VirtualBox, and Git (see [Requirements](#requirements)).

### Clone Bento

```bash
git clone https://github.com/chef/bento.git
cd bento
```

### Build a VirtualBox box

Bento's build layout has changed over time (older releases used per-OS JSON
templates such as `packer_templates/ubuntu/ubuntu-22.04-amd64.json`, newer ones
use HCL). Check the `README.md` in your cloned checkout for the exact command.
With the current HCL layout it looks like this:

```bash
packer init -upgrade ./packer_templates
packer build -only=virtualbox-iso.vm \
  -var-file=os_pkrvars/ubuntu/ubuntu-24.04-x86_64.pkrvars.hcl \
  ./packer_templates
```

The finished `.box` file is written to the `builds/` directory.

### Add your built box to Vagrant

```bash
vagrant box add --name my/ubuntu-24.04 builds/<file>.box
```

Then use `config.vm.box = "my/ubuntu-24.04"` in your Vagrantfile.

## Everyday commands

| Command | What it does |
| --- | --- |
| `vagrant up [name]` | Create/start the VM(s) |
| `vagrant ssh [name]` | SSH into a VM |
| `vagrant status` | Show VM states |
| `vagrant halt [name]` | Shut down gracefully |
| `vagrant reload [name]` | Restart and re-apply the Vagrantfile |
| `vagrant provision [name]` | Re-run provisioners |
| `vagrant suspend` / `vagrant resume` | Pause and resume |
| `vagrant destroy -f [name]` | Delete the VM(s) |
| `vagrant global-status --prune` | List all VMs and clean stale entries |

## Troubleshooting

**SSH: "Connection reset. Retrying..." during `vagrant up`**
The guest is usually still booting or starved of resources. Try fewer vCPUs and
less RAM per VM, start VMs one at a time, and raise the timeout with
`config.vm.boot_timeout = 600`. Set `vb.gui = true` to watch the console.

**Guest Additions version mismatch warning**
Harmless in most cases. Upgrade VirtualBox on the host, disable the shared
folder with `config.vm.synced_folder ".", "/vagrant", disabled: true`, or set
`vb.check_guest_additions = false`.

**VMs are slow or hang at boot on Linux**
KVM modules can conflict with VirtualBox. Check with `lsmod | grep kvm` and
unload them if you don't need KVM: `sudo modprobe -r kvm_intel kvm`
(or `kvm_amd`).

**Host-only network error: "IP address not within the allowed ranges"**
VirtualBox 7 restricts host-only ranges. Allow yours in `/etc/vbox/networks.conf`:

```bash
sudo mkdir -p /etc/vbox
echo "* 192.168.70.0/24" | sudo tee /etc/vbox/networks.conf
```

**Bridged networking over Wi-Fi is unreliable**
Some access points drop traffic from extra MAC addresses, and static IPs must
match your router's subnet. Prefer `private_network` (host-only) for lab
clusters.

**Speed up multi-VM setups**
Use `vb.linked_clone = true`, `config.vm.box_check_update = false`, and 1 vCPU
per VM unless a workload needs more.

## Security notes

- Boxes ship with the default `vagrant` user and an insecure key pair. Vagrant
  replaces the key on first boot, but the default password is still `vagrant`.
- If a VM uses a bridged (`public_network`) adapter, it is reachable from your
  real network. Change the default password and consider disabling SSH password
  authentication in a provisioner.
- Only download boxes from publishers you trust, and check their source.