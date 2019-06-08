# Summary
This guide demonstrates the following:
- SSH one-time password access management
- SSH certificate authority access management

## vagrant-aws Branch Available
vagrant-aws branch has been created to use the same script to deploy to AWS instead of local.
**NOT TO BE MERGED WITH MASTER**

## Prerequisites
1. Vagrant installed
1. VirtualBox installed

## Vagrant servers provisioned
Vault (consul backend) - Private IP - 192.168.50.100
Client - Private IP - 192.168.50.101
OTP - Private IP - 192.168.50.102

## Overview 

### Vault SSH One-Time Password
![](https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_otp_setup.png)
![](https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_otp_usage.png)

### Vault SSH Certificate Authority
![](https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_ca_setup.png)
![](https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_ca_usage.png)

## Provision steps
Within this working directory, execute the following. You must have access to create folders on your local dir as Ansible will need to do this to unseal Vault.

```
vagrant up
#or
sudo vagrant up
```
1. Set Vault address environment variable
```
export VAULT_ADDR='http://127.0.0.1:8200'
```
2. Login in to Vault Vagrant server 
```
Vault login <Admin Token>
```
### SSH One-Time Password

### Set up SSH backend and opt_key_role
1. Enable SSH backend for Vault provide to secure authentication and authorization for access to machines via the SSH :
```
vault secrets enable ssh
```
Create a role that has access to interact with SSH and request a one time password:
```
vault write ssh/roles/otp_key_role key_type=otp default_user=vagrant cidr_list=0.0.0.0/0
```
2. Create a policy that allows the authenticated person (either via a token or userpass) to request One time password from vault.
```
vault policy write admin -<<EOF
path "sys/mounts" {
  capabilities = ["list","read"]
}
path "secret/*" {
  capabilities = ["list", "read"]
}
path "secret/me" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "supersecret/*" {
  capabilities = ["list", "read"]
}
path "ssh-client-signer/*" {
  capabilities = ["read","list","create","update"]
}
path "aws/*" {
  capabilities = ["read","list","create","update"]
}
path "ssh/*" {
  capabilities = ["read","list","create","update"]
}

path "ssh/creds/otp_key_role" {
  capabilities = [ "update" ]
}
EOF
```

##Enable Userpass Method for "Engineer to login" - optional
Create a user password method to authenticate to Vault. It's never a good idea to login with the root key
```
vault auth enable userpass
```
Create a new user for the "Engineer to login in with, in this case we are calling them vault with a password of vault
```
vault write auth/userpass/users/vault password=vault policies=admin
```

### Request a One Time Password - Using the CLI
This can be either done with the root key (not normally recommended in prod) or logged in as the new user.
To login to vault (on the Vault server) as the new user.
```
vault login -method=userpass username=vault password=vault
```
Create One Time Password Credentials with the Server IP of the OTP server.
```
vault write ssh/creds/otp_key_role ip=192.168.50.102
```

##SSH to Server via OTP
Once at that terminal, issue the following command and enter the copied one-time password when prompted
```
ssh vagrant@192.168.50.102
```

The SSH connection should be successful. Any further attempts will fail, as the password is good for one use only!


### Request a One Time Password - Using the UI
Once Vagrant has provisioned the environment, you can login to the Web UI (At http://localhost:8200/ui/vault/auth) with `username: vault` and `password: vault`


### SSH One-Time Password usage

1. Login to the Vault web UI
1. Navigate to Secrets, then ssh, then `otp_key_role`
1. Click on the `opt_key_role`
1. Populate it with username 'vagrant' and host '192.168.50.102'. 
<img src="https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_otp_generate_creds_input.png" alt="otp generate creds" width="300">
1. Click 'Generate' and then you can click 'Copy Credentials' for ease of use.  
<img src="https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_otp_generate_creds_output.png" alt="copy creds" width="300">

On the commandline issue the following command to login to our example client

```
vagrant ssh client
```
Once at that terminal, issue the following command and enter the copied one-time password when prompted

```
ssh vagrant@192.168.50.102
```

The SSH connection should be successful. Any further attempts will fail, as the password is good for one use only!

### SSH Certificate Authority

On the Client virtual machine login to vault (if you haven’t already)
```
vault login -method=userpass username=vault password=vault
```
On the CA virtual machine login to vault with the admin token
```
Vault login <Admin Token>
```
## Host Key Signing
# Signing Key Configuration

1. Mount the secrets engine. For the most security, mount at a different path from the client signer (ssh must be enabled first `vault secrets enable ssh`)
```
vault secrets enable -path=ssh-host-signer ssh
```
2. Configure Vault with a CA for signing host keys using the /config/ca endpoint
```
vault write ssh-host-signer/config/ca generate_signing_key=true
```
3. Extend Host Key’s TLS
```
vault secrets tune -max-lease-ttl=87600h ssh-host-signer
```
4. Create a role for signing host keys (allow signed by user)
```
vault write ssh-host-signer/roles/hostrole \
    key_type=ca \
    ttl=87600h \
    allow_host_certificates=true \
    allowed_domains="localdomain,example.com" \
    allow_subdomains=true
```

5. Sign the host's SSH public key on the host (CA virtual machine)
```
vault write ssh-host-signer/sign/hostrole \
    cert_type=host \
    public_key=@/etc/ssh/ssh_host_rsa_key.pub
```
6. Set the resulting signed certificate as a hostcertificate:
```
vault write -field=signed_key ssh-host-signer/sign/hostrole \
    cert_type=host \
    public_key=@/etc/ssh/ssh_host_rsa_key.pub > /etc/ssh/ssh_host_rsa_key-cert.pub
```
(Or it can be done via batch))
```
sudo cat /etc/ssh/ssh_host_rsa_key.pub | vault write -format=json ssh-host-signer/sign/hostrole public_key=- cert_type=host | jq -r ".data.signed_key" | sudo tee /etc/ssh/ssh_host_rsa_key-cert.pub
Set permissions on the certificate sudo chmod 0640 /etc/ssh/ssh_host_rsa_key-cert.pub
```
7. Restart SSH
```
sudo systemctl restart sshd
```
8. On the Client virtual machine: Get the host signing CA public key
```
curl http://192.168.50.100:8200/v1/ssh-host-signer/public_key
```
9. Cat out key and pasted it in to /home/vagrant/.ssh/known_hosts
```
sudo echo '@cert-authority *.example.com <trusted-host-keys.pem>’ > /home/vagrant/.ssh/known_hosts
```
10. Restart SSHD
```
sudo systemctl restart sshd
```
## Client Key Signing
#Signing Key & Role Configuration
1. Mount a backend's instance for signing client keys.
```
vault secrets enable -path=ssh-client-signer ssh
```
2. Configure Vault with a CA for signing client keys using the /config/ca endpoint
```
vault write ssh-client-signer/config/ca generate_signing_key=true
```
This will provide an ssh key.
3. ON THE HOST (CA server)- Copy the key on to the host server using vaults ca endpoint via API. 
```
sudo curl -o /etc/ssh/trusted-user-ca-keys.pem http://192.168.50.100:8200/v1/ssh-client-signer/public_key
```
4. Restart the SSH service to pick up the changes.
```
sudo systemctl restart sshd
```
5. ON THE Vault Server - Create a named Vault role for signing client keys.
```
vault write ssh-client-signer/roles/clientrole -<<"EOH"
{
  "allow_user_certificates": true,
  "allowed_users": "*",
  "default_extensions": [
    {
      "permit-pty": ""
    }
  ],
  "key_type": "ca",
"key_id_format": "vault-{{role_name}}-{{token_display_name}}-{{public_key_hash}}",
  "default_user": "vagrant",
  "ttl": "30m0s"
}
EOH
```
# Client SSH Authentication (on Client virtual machine)
Generate SSH keys if needed:
```
ssh-keygen -f /home/vagrant/.ssh/id_rsa -t rsa -N ''
```
# Using the CLI:
1. Ask Vault to sign your public key
```
vault write ssh-client-signer/sign/clientrole \
    public_key=@$HOME/.ssh/id_rsa.pub
```
2. Save the resulting signed, public key to disk.
```
vault write -field=signed_key ssh-client-signer/sign/clientrole \
    public_key=@$HOME/.ssh/id_rsa.pub > ~/.ssh/signed-cert.pub
```
# Using the UI:
1. Login to the Vault web UI
2. Copy the vagrant user's public key to the clipboard `cat ~/.ssh/id_rsa.pub`
![](https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_ca_copy_pubkey.png)
3. Paste it into the web UI at the 'Public Key' field
![](https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_ca_paste_pubkey.png)
4. Click Sign
5. Click the 'Copy key' button to copy the signed public key to the clipboard
![](https://raw.githubusercontent.com/hashicorp/vault-guides/master/assets/vault_ssh_ca_signed_pubkey.png)
6. Within the client connection terminal, save the signed key to `~/.ssh/id_rsa-cert.pub` with the following command `echo '<paste key>' > ~/.ssh/id_rsa-cert.pub`

# Connect to the server configured for SSH CA using the following command:
```
ssh -i ~/.ssh/signed-cert.pub -i ~/.ssh/id_rsa vagrant@192.168.50.103
```
## Tear Down Steps
Execute the following to destroy the environment. 
```
vagrant destroy -f
```