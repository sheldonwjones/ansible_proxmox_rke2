## Create the VM template

Complete this section on the proxmox server.

### Create an ssh key for the ansible user

Run ssh-keygen and create a key without a password. We will need the `ansible.key` file when running `ansible-playbook`.
```bash
ssh-keygen -f ansible.key
```

### Download and customize the cloud image

Download the Ubuntu 22.04 jammy cloud image.

```bash
# rm -f jammy-server-cloudimg-amd64.img
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

Use virt-customize to customize the image. You may need to install it using `apt-get install libguestfs-tools`

- this will run apt update on the image
- install qemu-guest-agent
- create the ansible user and copy the ansbile.key.pub ssh key to the ansible users authorized keys
- give the ansible user sudo access without a password
- setup a uniq machine id

```bash
echo "ansible ALL=(ALL) NOPASSWD: ALL" > ansible_sudo
virt-customize -a jammy-server-cloudimg-amd64.img --update
virt-customize -a jammy-server-cloudimg-amd64.img --install qemu-guest-agent
virt-customize -a jammy-server-cloudimg-amd64.img --run-command 'useradd --shell /bin/bash ansible'
virt-customize -a jammy-server-cloudimg-amd64.img --run-command 'mkdir -p /home/ansible/.ssh'
virt-customize -a jammy-server-cloudimg-amd64.img --ssh-inject ansible:file:ansible.key.pub
virt-customize -a jammy-server-cloudimg-amd64.img --run-command 'chown -R ansible:ansible /home/ansible'
virt-customize -a jammy-server-cloudimg-amd64.img --upload ansible_sudo:/etc/sudoers.d/ansible
virt-customize -a jammy-server-cloudimg-amd64.img --run-command 'chmod 0440 /etc/sudoers.d/ansible'
virt-customize -a jammy-server-cloudimg-amd64.img --run-command 'chown root:root /etc/sudoers.d/ansible'
virt-customize -a jammy-server-cloudimg-amd64.img --run-command '>/etc/machine-id'
```

### Create the proxmox template

The script below assumes that your template will be using the `local-zfs` datastore and the template ID will be `9000`. By default this template will provistion VMs with 2G of ram, 2 CPU cores, and dhcp networking.

```
qm create 9000 --name "ubuntu-22.04-cloudinit-template" --ostype l26 \
    --memory 2048 \
    --cpu host --socket 1 --cores 2 \
    --agent 1 \
    --bios ovmf --machine q35 --efidisk0 local-zfs:0,pre-enrolled-keys=0 \
    --net0 virtio,bridge=vmbr0
qm importdisk 9000 jammy-server-cloudimg-amd64.img local-zfs
qm set 9000 --scsihw virtio-scsi-pci --virtio0 local-zfs:vm-9000-disk-1,discard=on
qm set 9000 --boot order=virtio0
qm set 9000 --ide2 local-zfs:cloudinit
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
qm set 9000 --sshkeys ~/.ssh/authorized_keys
qm set 9000 --ipconfig0 ip=dhcp
qm template 9000
```

## Ansbile

Complete the following steps on the machine where you'll be running ansible.

### Setup

Clone this repos and and add the `ansbile.key` file from above.

```
❯ tree
.
├── ansible.cfg
├── ansible.key
├── demo.yaml
├── inventories
│   └── demo.yaml
├── playbooks
│   ├── create_vm.yaml
│   ├── delete_vm.yaml
│   ├── jupyterhub.yaml
│   ├── longhorn.yaml
│   └── rke2.yaml
├── README.md
└── requirements.yaml
```

Install the ansible requirements.

```
ansible-galaxy install --force -r requirements.yaml
```

### Update the inventory `inventories/demo.yaml`

Change the ip addresses to match your network requirements and make sure `vmid` matches the template ID that you created in the above steps.

Update the proxmox server settings to match your proxmox environment.

```yaml
proxmox:
  api_user: root@pam
  api_token_secret: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  api_token_id: ci
  api_host: 192.168.x.y
  node: xxxxxx
```

### Run the demo playbooks

The demo playbook will create 6 VMs each using 3G of RAM with 4 CPU cores. They will each be given 20G of harddrive space and have static IPs assigned. These settings are all provided by the `inventories/demo.yaml` inventory file. Once the cluster has been created the playbook installs longhorn and sets up jupyterhub via helm.

```
ansible-playbook demo.yaml
```

This assumes that the ssh key thet we created in the above step is located in a file called `ansible.key`. You can change this in the `ansible.cfg` file.
