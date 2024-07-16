# Table of Contents
- Install Dependencies
- Set up SSH Access for All Nodes
- Disable Firewall
- Clone and Prepare Kubespray
- Set up Kubespray Inventory
- Deploy Kubernetes with Kubespray

# Install Dependencies
```console
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible git python3-pip python3-venv
sudo apt install expect
```
This stage installs necessary dependencies, including Ansible, Git, Python, and Expect.

# Set up SSH Access for All Nodes
```console
sudo rm -f /home/azureagent/.ssh/vm_private_key
# echo $(vmprivatekey) >> /home/azureagent/.ssh/vm_private_key
cat << 'EOF' > /home/azureagent/.ssh/vm_private_key
$(vmprivatekey)
EOF
chmod 600 /home/azureagent/.ssh/vm_private_key
chown root:root /home/azureagent/.ssh/vm_private_key
# It is required that your private key files are NOT accessible by others.
ips=($(ipsvar))

eval "$(ssh-agent -s)"
echo "SSH_AGENT_PID=$SSH_AGENT_PID" >> ~/.ssh/environment
echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK" >> ~/.ssh/environment
source ~/.ssh/environment

# Create the expect script
cat << 'EOF' > add_ssh_key.exp
#!/usr/bin/expect -f
set timeout -1
set key_path [lindex $argv 0]
set passphrase [lindex $argv 1]
spawn ssh-add $key_path
expect "Enter passphrase for"
send "$passphrase\r"
expect eof
EOF

# Make the script executable
chmod +x add_ssh_key.exp

# Run the expect script with the SSH key and passphrase
./add_ssh_key.exp /home/azureagent/.ssh/vm_private_key "$(vmpassphrase)"
```
This step sets up SSH access for all nodes using a provided private key. 

# Disable Firewall
```console
sudo swapoff -a
sudo ufw disable
sudo systemctl disable ufw
```
This stage disables the firewall and swap to ensure smooth Kubernetes installation and operation.

# Clone and Prepare Kubespray
```console
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray/
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt
pip3 install ruamel.yaml
```
This stage clones the Kubespray repository and prepares the environment by installing required Python packages.

# Set up Kubespray Inventory
```console
mkdir -p kubespray/inventory/$(clusternamevar)
cp -r kubespray/inventory/sample/* kubespray/inventory/$(clusternamevar)/
source kubespray/venv/bin/activate
declare -a IPS=($(ipsvar))
CONFIG_FILE=kubespray/inventory/$(clusternamevar)/hosts.yaml python3 kubespray/contrib/inventory_builder/inventory.py ${IPS[@]}
```
This step sets up the Kubespray inventory using the provided IPs. It creates a new inventory directory and copies the sample inventory files.

# Deploy Kubernetes with Kubespray
```console
cd kubespray
source venv/bin/activate
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook -i inventory/$(clusternamevar)/hosts.yaml -u root --private-key=/home/azureagent/.ssh/vm_private_key --become --become-user=root cluster.yml
```
This final stage deploys Kubernetes using Kubespray with Ansible. It runs the playbook to set up the cluster using the specified inventory and SSH key.



# 
```console

```