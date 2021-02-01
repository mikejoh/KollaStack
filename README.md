# KollaStack

KollaStack is aimed to be an alternative all-in-one deployment of OpenStack using your local computer as deploy host using `kolla-ansible`.

KollaStack consists of the following components:
* [`kolla-ansible`](https://docs.openstack.org/kolla-ansible/latest/)
* [OpenStack](https://www.openstack.org/)
* [Vagrant](https://www.vagrantup.com/)
* [VirtualBox](https://www.virtualbox.org/)

OpenStack services available out-of-the-box:
* Nova (compute)
* Neutron (network)
* Heat (orchestration)
* Keystone (identity)
* Glance (images)
* Cinder (block storage - via LVM)
# Local environment

Resources needed:
* `8192MB` RAM
* `2` vCPUs
* `30GB` Disk space (15GB for the host root disk and 15GB used by Cinder to provision storage)

Tested the following OpenStack release(s):
- [X] `Victoria` and `kolla-ansible` version `11.0.0`
- [ ] `Ussuri`
- [ ] `Train`
- [ ] `Stein`

Tested deploying OpenStack on the following VM host OS's:
- [X] `Ubuntu 18.04`
- [ ] `Ubuntu 20.04`

Tested on the following VirtualBox version(s):
- [X] `6.1.10`

Tested with the following Vagrant version(s):
- [X] `2.2.14`

# Step-by-step instructions

_**IMPORTANT:** Before starting please make any necessary changes to the Vagrant VM configuration file ([`vagrant.yml`](kollastack/vagrant.yml))._

1. Create a local `venv`, source it and install required Python packages:
```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
2. Create a local configuration directory that'll contain a global `kolla-ansible` configuration file. *Please use the provided example file if you want to, if so just rename it to `globals.yml` and skip this step!*
```
mkdir -p ./etc/kolla
cp .venv/share/kolla-ansible/etc_examples/kolla/globals.yml ./etc/kolla
```

The `globals-example.yml` file contains the following configured options:
```
config_strategy: "COPY_ALWAYS"
node_custom_config: "./etc/kolla/config"
enable_haproxy: "no
enable_openstack_core: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
cinder_volume_group: "cinder-volumes"
kolla_base_distro: "ubuntu"
kolla_install_type: "binary"
openstack_release: "victoria"
kolla_internal_vip_address: "192.168.50.10"
kolla_external_vip_address: "{{ kolla_internal_vip_address }}"
network_interface: "enp0s8"
neutron_external_interface: "enp0s9"
neutron_plugin_agent: "openvswitch"
nova_compute_virt_type: "qemu"
```
_Note that there's hundreds of other things to tweak and configure in this file, please do test and see what's possible, the provided `globals-example.yml` is more of a bare minimum config file that will give you a working OpenStack deployment in one VM._

_Note that i've added a custom [`ml2_conf.ini`](etc/kolla/config/neutron/ml2_conf.ini) to enable creation of various types of networks in `neutron`._

3. Copy the `passwords.yml` to the local `kolla-ansible` configuration directory and generated all the passwords:
```
cp .venv/share/kolla-ansible/etc_examples/kolla/passwords.yml ./etc/kolla
kolla-genpwd -p ./etc/kolla/passwords.yml
```

4. Start the VM in VirtualBox:
```
cd kollastack/ && vagrant up
```

5. With your venv still sourced run the following from the root of this repository, the `all-in-one` inventory should be configured correctly for you:
```
kolla-ansible --configdir ./etc/kolla -i all-in-one bootstrap-servers
kolla-ansible --configdir ./etc/kolla -i all-in-one prechecks
kolla-ansible --configdir ./etc/kolla -i all-in-one deploy
``` 

6. Create a NAT Network in VirtualBox via File > Preferences > Network. Give the network the name `extnet` to match what we defined in the [`Vagrantfile`](kollastack/Vagrantfile) for network adapter 3.

7. Generate the `admin` credentials file by running:
```
kolla-ansible post-deploy
``` 

8. Login to Horizon as `admin` here: [https://192.168.50.10](https://192.168.50.10). Use the generated password called `keystone_admin_password` in the [`passwords.yml`](etc/kolla/passwords.yml) file.

## [WIP] Create VM flavor and image

1. Create a VM flavor, we'll be testing with a small CirrOS instance:
```
openstack --os-compute-api-version 2.55 flavor create --ram 256 --disk 1 --public m1.super.small
``` 

2. Download a CirrOS image and upload it to the Glance image service:
```
openstack image create --public --container-format bare --disk-format qcow2 --file <PATH TO IMG> cirros
```

3. OPTIONAL: Create a keypair using your public SSH key, not needed for the CirrOS test but in future scenarios:
```
openstack keypair create --public-key <PATH TO PUBLIC KEY> <KEYPAIR NAME>
``` 
## [WIP] Create external network and subnet 
1. Create an external network:
```
openstack network create --external --provider-physical-network physnet1 --provider-network-type flat ext-net
```

2. Create a subnet in the network above:
```
openstack subnet create --no-dhcp --allocation-pool start=10.0.2.15,end=10.0.2.250 --dns-nameserver 8.8.8.8 --network ext-net --subnet-range 10.0.2.0/24 --gateway 10.0.2.2 ext-subnet
```

## [WIP] Launch an instance

7. Example to launch a CirrOS instance:
```
openstack server create \
  --image <IMAGE NAME> \
  --flavor m1.tiny \
  --key-name <KEYPAIR NAME> \
  --network ext-net \
    cirros
```

# Todo

* Reachable instances with FLIPs
* Octavia (Load Balancing)