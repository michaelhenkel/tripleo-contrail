# tripleo-contrail


1. Log in to machine, ensure it's up to date, install git and $tools    
```
sudo yum -y update
sudo yum install vim-enhanced git wget
```
2. Configure host repos (using tripleo.sh)    
```
git clone https://github.com/openstack-infra/tripleo-ci.git
tripleo-ci/scripts/tripleo.sh --repo-setup
```
2. Configure undercloud VM    
```
sudo yum install -y instack-undercloud

# instack-virt-setup can't be run as root
sudo useradd stack
sudo passwd stack  # specify a password
echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
sudo chmod 0440 /etc/sudoers.d/stack
su - stack

# Note KSM is enabled by default on centos7 so you can overcommit somewhat
export NODE_CPU=4
export NODE_MEM=8096
export UNDERCLOUD_NODE_CPU=2
export UNDERCLOUD_NODE_MEM=8096
export NODE_COUNT=10

instack-virt-setup

scp /home/stack/.cache/image-create/CentOS-7-x86_64-GenericCloud.qcow2.xz root@<undercloud VM IP>:/tmp
```

3. Install Undercloud on the VM we just prepared    
```
ssh root@<undercloud VM IP>
su - stack
mkdir -p /home/stack/.cache/image-create
cp /tmp/CentOS-7-x86_64-GenericCloud.qcow2.xz /home/stack/.cache/image-create/CentOS-7-x86_64-GenericCloud.qcow2.xz
git clone https://github.com/openstack-infra/tripleo-ci.git
export STABLE_RELEASE=mitaka
bash -x tripleo-ci/scripts/tripleo.sh --repo-setup
yum repolist
bash -x tripleo-ci/scripts/tripleo.sh --undercloud


# See if heat is running OK
. stackrc
heat stack-list
```

4. Build the overcloud images    
```
bash -x export OVERCLOUD_IMAGES_ARGS='--type overcloud-full --builder-extra-args overcloud-contrail-neutron' && tripleo-ci/scripts/tripleo.sh --overcloud-images
glance image-list
```

5. Register nodes    
```
bash -x tripleo-ci/scripts/tripleo.sh --register-nodes
#(Calls openstack baremetal import --json /home/stack/instackenv.json)
ironic node-list
```
6. Create Contrail Controller image    
```
ramdisk_id=$(glance image-show overcloud-full-initrd | awk '/\| id/  {print $4}')
kernel_id=$(glance image-show overcloud-full-vmlinuz | awk '/\| id/  {print $4}')
glance image-create --name contrail --disk-format qcow2 --container-format bare --property kernel_id=$kernel_id --property ramdisk_id=$ramdisk_id --file overcloud-contrail-3.0.2-51.qcow2
ironic node-list
```

7. Deploy overcloud    
```
openstack overcloud deploy --templates ~/templates \
	-e ~/templates/environments/network-isolation.yaml \
	-e ~/templates/my_deployment.yaml \
	-e ~/templates/environments/neutron-opencontrail.yaml \
	-e ~/templates/environments/contrail.yaml \
	--control-scale 1 \
	--compute-scale 1 \
	--ceph-storage-scale 0 \
	--block-storage-scale 0 \
	--swift-storage-scale 0 \
	--neutron-network-type vxlan \
	--neutron-tunnel-types vxlan \
	--libvirt-type qemu \
	--ntp-server pool.ntp.org
```


