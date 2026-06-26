# IPFS & IPFS Cluster Ansible

Automated deployment and configuration of a private IPFS network and IPFS Cluster using Ansible.
## Features
- Install and configure a private IPFS
- Install and configure IPFS Cluster for pin management
- Secure secret management with Ansible Vault
- Dedicated system user for IPFS services

## Requirements

- Ansible >= 2.9
- SSH access to target hosts
- Sudo privileges on target hosts
## Repository Structure

```
Ansible/
├── site.yml                              # Main playbook 
├── ipfs.yml                              # Playbook IPFS only
├── ipfs-cluster.yml                      # Playbook Cluster only
├── inventory.yml                         # Inventory (IPs/hostnames)
├── README.md                           
├── group_vars/
│   ├── ipfs/
│   │   ├── main.yml                     # IPFS Configuration 
│   │   └── vault.yml                    # SECRET: vault_ipfs_swarm_key (encrypted)
│   └── ipfs-cluster/
│       ├── main.yml                     # Configuration Cluster
│       └── vault.yml                    # SECRET: vault_cluster_secret (encrypted)
├── host_vars/
│   ├── node0.yml                        # Config spécifique
│   ├── node1.yml
│   └── ...
└── roles/
    ├── ipfs/                             # Role installation IPFS
    │   ├── tasks/
    │   ├── templates/
    │   └── handlers/
    └── ipfs-cluster/                     # Role installation Cluster
        ├── tasks/
        ├── templates/
        └── handlers/
```

## Inventory Configuration

Edit `inventory.yml` and replace the example hosts with your own infrastructure.

Example:

```yaml
ipfs_cluster:
  hosts:
    node0:
      ansible_host: 192.168.1.10
    node1:
      ansible_host: 192.168.1.11
    node2:          
      ansible_host: 192.168.1.12
    node3:          
      ansible_host: 192.168.1.13

```

## Variables

General IPFS configuration is stored in:

```text
group_vars/ipfs/main.yml
```

General IPFS Cluster configuration is stored in:

```text
group_vars/ipfs-cluster/main.yml
```

Host-specific variables may be defined in:

```text
host_vars/<hostname>.yml
```

## Secret Management

This project uses Ansible Vault to protect sensitive information.

#### IPFS Swarm Key

1. Generate a valid IPFS swarm key using the official tool:
```bash
go install github.com/Kubuxu/go-ipfs-swarm-key-gen/ipfs-swarm-key-gen@latest
```

2. Create the swarm key:
```bash
ipfs-swarm-key-gen > swarm.key
```

3. Encrypt it:
```bash
ansible-vault encrypt_string --vault-id ipfs_dev@prompt '$(cat swarm.key)' --name 'vault_ipfs_swarm_key'
```

4. edit and paste the encrypted output inside:
```text
group_vars/ipfs/vault.yml
```


Example:

```yaml
vault_ipfs_swarm_key: !vault |
          $ANSIBLE_VAULT;1.2;AES256;ipfs_dev
          ...
```

#### IPFS Cluster Secret

1. Generate a cluster secret:

Use
```bash
openssl rand -hex 32
```
or

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

2. Encrypt it:

```bash
ansible-vault encrypt_string --vault-id cluster_dev@prompt '<cluster-secret>' --name 'vault_cluster_secret'
```

3. edit and paste the encrypted output inside:
```text
group_vars/ipfs-cluster/vault.yml
```

Example:

```yaml
vault_cluster_secret: !vault |
          $ANSIBLE_VAULT;1.2;AES256;cluster_dev
          ...
```

#### Vault Password Files (Optional)

Instead of entering vault passwords interactively, password files may be used.

Example:

```bash
echo "my-ipfs-password" > .vault_ipfs_dev
echo "my-cluster-password" > .vault_cluster_dev

chmod 600 .vault_ipfs_dev
chmod 600 .vault_cluster_dev
```

Or use`--vault-id` with `@prompt` to enter the passwd interactively
 
```bash
ansible-vault encrypt_string "$SWARM_KEY" -vault-id ipfs_dev@prompt --name 'vault_ipfs_swarm_key'
```

```bash
ansible-vault encrypt_string "<cluster-secret>" --vault-id cluster_dev@prompt --name 'vault_cluster_secret'
```
##  Deployment

### Password in a file

```bash
ansible-playbook site.yml -i inventory.yml --vault-id ipfs_dev@.vault_ipfs_dev --vault-id cluster_dev@.vault_cluster_dev -u <ssh-user>
```

### Password interactive

```bash
ansible-playbook site.yml -i inventory.yml --vault-id ipfs_dev@prompt --vault-id cluster_dev@prompt -u <ssh-user>
```
### Other options 

#### Deploy everything
``` bash
ansible-playbook site.yml -i inventory.yml --vault-id ipfs_dev@prompt --vault-id cluster_dev@prompt -u <ssh-user>
```
#### Deploy IPFS only
```bash
ansible-playbook ipfs.yml -i inventory.yml --vault-id ipfs_dev@prompt -u <ssh-user>
```
#### Deploy IPFS Cluster only
```bash
ansible-playbook ipfs-cluster.yml -i inventory.yml --vault-id cluster_dev@prompt -u <ssh-user>
```
#### Deploy a specific host
```bash
ansible-playbook site.yml -i inventory.yml --vault-id ipfs_dev@prompt --vault-id cluster_dev@prompt -u <ssh-user> --limit node1
```
#### Verbose mode
```bash
ansible-playbook site.yml -i inventory.yml --vault-id ipfs_dev@prompt --vault-id cluster_dev@prompt -u <ssh-user> -vv
```

### Password-Based Authentication

If SSH keys and passwordless sudo are not configured:

```bash
ansible-playbook site.yml -i inventory.yml --vault-id ipfs_dev@prompt --vault-id cluster_dev@prompt -u <ssh-user> -k -K
```
Where:

- `-k` asks for the SSH password
    
- `-K` asks for the sudo password
    
## Why Create a Dedicated `ipfs` User?

This playbook automatically creates:

- An ipfs user with no interactive shell (/bin/false)
- An **`ipfs` group** used for file ownership and permissions

### Benefits

✔  **Security isolation:** IPFS services do not run as root

✔  **Least privilege:** The ipfs user owns and operates /var/lib/ipfs, limiting access for non-privileged users

✔ **Process management:** IPFS and IPFS Cluster services run under the `ipfs` account

✔  **Data protection:** Sensitive files such as swarm keys are protected with restrictive permissions

✔ **Easier maintenance:** File ownership and permissions remain consistent across deployments


## Data Directory Layout

All IPFS and IPFS Cluster data is stored under:

```text
/var/lib/ipfs/
├── .ipfs/
│   ├── config
│   ├── datastore
│   └── swarm.key
│   └── ...
└── .ipfs-cluster/
│   ├── service.json
│   ├── peerstore
│   └── identity.json
│   └── ...
└── ...
```


## Using IPFS Commands

Because the `ipfs` account has no interactive shell, administrators may execute IPFS commands through `sudo`.

### Optional Shell Aliases

Add the following aliases to your shell configuration file :

```bash
alias ipfs="sudo -u ipfs IPFS_PATH=/var/lib/ipfs/.ipfs ipfs"
alias ipfs-cluster-ctl="sudo -u ipfs IPFS_PATH=/var/lib/ipfs/.ipfs-cluster ipfs-cluster-ctl"
```

Reload your shell:

```bash
source ~/.bashrc
```

### Examples
#### List swarm peers

```bash
ipfs swarm peers
```

#### Display IPFS configuration
```bash
ipfs config show
```
#### Add a file
```bash
ipfs add /tmp/file.txt
```
#### List cluster peers
```bash
ipfs-cluster-ctl peers
```
#### Show cluster status
```bash
ipfs-cluster-ctl status
```
#### Display cluster information
```bash
ipfs-cluster-ctl info
```

## Troubleshooting

### Vault Decryption Failed

Verify that the vault IDs used during execution match the labels used when encrypting secrets.

Example:

```bash
--vault-id ipfs_dev@prompt
--vault-id cluster_dev@prompt
```

### Check IPFS Service

```bash
sudo systemctl status ipfs
sudo journalctl -u ipfs -f
```

### Check IPFS Cluster Service

```bash
sudo systemctl status ipfs-cluster
sudo journalctl -u ipfs-cluster -f
```

## Security Checklist

✔  Secrets stored in Ansible Vault
        
✔  Dedicated service account for IPFS
    
✔  Private swarm key shared across all cluster nodes
    
✔  Shared cluster secret across all cluster nodes
    
✔  SSH key authentication recommended
    
✔ Least-privilege permissions applied to deployed files
    

## Resources

- [Ansible Vault Documentation](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [IPFS Documentation](https://docs.ipfs.io/)
- [IPFS Cluster Documentation](https://cluster.ipfs.io/)
- [Filesystem Hierarchy Standard (FHS)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html)


