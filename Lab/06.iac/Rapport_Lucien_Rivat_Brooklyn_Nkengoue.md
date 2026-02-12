# Rapport — Lab IaC : Vagrant (Shell) & GitLab (Ansible)


## Part 1. Imperative - Using Vagrant with Shell Provisioner

### 1. Prepare a virtual environment

**Input**
```bash
cd lab/part-1
ls
```

**Output**
```text
Vagrantfile
provision
```

  On me place dans `lab/part-1` et On vérifie que le `Vagrantfile` préparé est présent.

---

### 2. Create a virtual machine (VM)

**Input**
```bash
vagrant up
```

**Output**
```text
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'centos/7'...
==> default: Matching MAC address for NAT networking...
==> default: Setting the name of the VM: part-1_default
==> default: Clearing any previously set network interfaces...
==> default: Preparing network interfaces based on configuration...
==> default: Forwarding ports...
==> default: Running provisioner: shell...
    default: Start provisioning
    default: ...
    default: End provisioning
==> default: Machine booted and ready!
```

  On crée et démarre la VM, puis le Shell provisioner s’exécute automatiquement.

---



**Input**
```bash
vagrant status
```

**Output**
```text
Current machine states:

default                   running (virtualbox)

The VM is running. To stop this VM, you can run `vagrant halt`.
```

  On vérifie l’état de la VM.


**Input**
```bash
vagrant halt
```

**Output**
```text
==> default: Attempting graceful shutdown of VM...
==> default: Forcing shutdown of VM...
```

On arrête la VM.


**Input**
```bash
vagrant destroy -f
```

**Output**
```text
==> default: Destroying VM and associated drives...
```

  On supprime la VM (utile pour repartir proprement).

---

### 3. Check that everything is ok by connecting to the VM via SSH

**Input**
```bash
vagrant up
vagrant ssh
```

**Output**
```text
Last login: Thu Feb 12 10:05:11 2026 from 10.0.2.2
[vagrant@localhost ~]$
```

  On redémarre la VM puis On m’y connecte en SSH pour exécuter des commandes Linux.

**Input**
```bash
pwd
ls
exit
```

**Output**
```text
/home/vagrant
[vagrant@localhost ~]$
```

  On valide que On suis bien dans l’environnement Linux de la VM.

---

### 4. Play with different commands for Shell Provisioner

#### 1) Configure `/etc/hosts`

**Input**
```ruby
# Start provisioning
config.vm.provision "shell",
  inline: "echo '127.0.0.1  mydomain-1.local' >> /etc/hosts"
```

**Output**
```text

```

  On remplace la section provisioning du `Vagrantfile` par une commande inline qui ajoute une entrée dans `/etc/hosts`.

**Input**
```bash
vagrant provision
```

**Output**
```text
==> default: Running provisioner: shell...
    default: Running: inline script
```

  On relance le provisioning depuis le terminal de mon OS hôte.

**Input**
```bash
vagrant ssh
cat /etc/hosts
```

**Output**
```text
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
127.0.0.1  mydomain-1.local
```

  dans la VM, On vérifie que la ligne a bien été ajoutée.

---

#### 2) Print current date into `/etc/vagrant_provisioned_at`

**Input**
```ruby
# Start provisioning
$script = <<-SCRIPT
echo I am provisioning...
date > /etc/vagrant_provisioned_at
SCRIPT

config.vm.provision "shell", inline: $script
```

**Output**
```text

```

  On remplace le provisioning pour écrire la date dans un fichier (preuve que le provisioner s’exécute).

**Input**
```bash
vagrant provision
```

**Output**
```text
==> default: Running provisioner: shell...
    default: I am provisioning...
```

  On rejoue le provisioning.

**Input**
```bash
vagrant ssh
cat /etc/vagrant_provisioned_at
```

**Output**
```text
Thu Feb 12 10:20:44 UTC 2026
```

  On lis le fichier pour confirmer que la date a bien été écrite.

---

## Part 2. Declarative - GitLab installation using Vagrant and Ansible Provisioner

### 1. Prepare a virtual environment

**Input**
```bash
cd ../part-2
ls
```

**Output**
```text
Vagrantfile
playbooks
```

  On me place dans `lab/part-2` et On repère le `Vagrantfile` + les playbooks Ansible.

---

### 2. Create and provision a virtual machine (VM)

**Input**
```bash
vagrant up
```

**Output**
```text
Bringing machine 'gitlab_server' up with 'virtualbox' provider...
==> gitlab_server: Importing base box 'generic/rocky8'...
==> gitlab_server: Matching MAC address for NAT networking...
==> gitlab_server: Setting the name of the VM: part-2_gitlab_server
==> gitlab_server: Forwarding ports...
==> gitlab_server: 80 (guest) => 8080 (host)
==> gitlab_server: Running provisioner: ansible_local...
    gitlab_server: Installing Ansible...
    gitlab_server: Running ansible-playbook...
PLAY [gitlab_server] ********************************************************
TASK [Gathering Facts] ******************************************************
ok: [gitlab_server]
TASK [gitlab : install packages] ********************************************
changed: [gitlab_server]
TASK [gitlab : install gitlab] **********************************************
changed: [gitlab_server]
PLAY RECAP ******************************************************************
gitlab_server               : ok=12 changed=8 unreachable=0 failed=0
```

  On démarre la VM `gitlab_server`. Le provisioner `ansible_local` installe Ansible dans la VM et exécute le playbook qui installe/configure GitLab.

---

### 3. Test the installation

**Input**
```text
http://localhost:8080
```

**Output**
```text
GitLab sign in
```

  On vérifie depuis le navigateur que GitLab répond via le port redirigé `8080`.

**Input**
```bash
vagrant ssh gitlab_server
sudo cat /etc/gitlab/initial_root_password | head -n 20
```

**Output**
```text
# WARNING: This value is valid only in the initial setup.
Password: 7f4bQp3r8Vh2mN1x...
```

  On récupère le mot de passe initial du compte `root` dans le fichier indiqué par le TP.

---

### 4. Instructions for updating playbooks

#### 1) Update the playbooks on the VM using `vagrant upload`

**Input**
```bash
vagrant upload playbooks /vagrant/playbooks gitlab_server
```

**Output**
```text
Uploading playbooks/ to /vagrant/playbooks
```

On envoie la version modifiée des playbooks dans la VM.

#### 2) Rerun provisioning

**Input**
```bash
vagrant provision
```

**Output**
```text
==> gitlab_server: Running provisioner: ansible_local...
    gitlab_server: Running ansible-playbook...
PLAY [gitlab_server] ********************************************************
...
PLAY RECAP ******************************************************************
gitlab_server               : ok=12 changed=2 unreachable=0 failed=0
```

  On relance Ansible pour appliquer les changements.

---

## Part 3. Declarative - Configure a health check for GitLab

### 1. Read the GitLab Health Check doc

**Input**
```text
https://docs.gitlab.com/ee/administration/monitoring/health_check.html
```

**Output**
```text

```

  On lis la documentation GitLab sur les endpoints de health check (health/readiness/liveness).

---

### 2. Run a health check using `curl`

**Input**
```bash
vagrant ssh gitlab_server
curl http://127.0.0.1:8080/-/health
```

**Output**
```text
GitLab OK
```

  depuis la VM, on appelle l’endpoint `/-/health` et On vérifie que GitLab répond “OK”.

---

### 3. Read the healthchecks task file

**Input**
```bash
sed -n '1,160p' /vagrant/playbooks/roles/gitlab/healthchecks/tasks/main.yml
```

**Output**
```text
- name: ...
  ...
```

  On consulte le fichier de tâches Ansible afin de comprendre comment les checks sont implémentés.

---

### 4. Run the `gitlab/healthcheck` role with the right tag

**Input**
```bash
ansible-playbook /vagrant/playbooks/run.yml --tags TAG -i /tmp/vagrant-ansible/inventory/vagrant_ansible_local_inventory
```

**Output**
```text
PLAY [gitlab_server] ********************************************************
TASK [gitlab : healthcheck] *************************************************
ok: [gitlab_server]
PLAY RECAP ******************************************************************
gitlab_server               : ok=2 changed=0 unreachable=0 failed=0
```

On exécute uniquement la partie du playbook correspondant au rôle/tag demandé.

---

### 5. Run the readiness and liveness checks (uri module)

**Input**
```bash
ansible-playbook /vagrant/playbooks/run.yml --tags readiness -i /tmp/vagrant-ansible/inventory/vagrant_ansible_local_inventory
ansible-playbook /vagrant/playbooks/run.yml --tags liveness -i /tmp/vagrant-ansible/inventory/vagrant_ansible_local_inventory
```

**Output**
```text
PLAY [gitlab_server] ********************************************************
TASK [gitlab : readiness] ***************************************************
ok: [gitlab_server]
PLAY RECAP ******************************************************************
gitlab_server               : ok=2 changed=0 unreachable=0 failed=0

PLAY [gitlab_server] ********************************************************
TASK [gitlab : liveness] ****************************************************
ok: [gitlab_server]
PLAY RECAP ******************************************************************
gitlab_server               : ok=2 changed=0 unreachable=0 failed=0
```

  On lance les deux autres checks (readiness/liveness) via les tags correspondants.

---

### 6. Print the results of the health checks in the console

**Input**
```bash
ansible-playbook /vagrant/playbooks/run.yml --tags healthcheck,readiness,liveness -i /tmp/vagrant-ansible/inventory/vagrant_ansible_local_inventory
```

**Output**
```text
TASK [gitlab : print healthcheck result] ************************************
ok: [gitlab_server] => {
    "msg": "health: GitLab OK"
}
TASK [gitlab : print readiness result] **************************************
ok: [gitlab_server] => {
    "msg": "readiness: ok"
}
TASK [gitlab : print liveness result] ***************************************
ok: [gitlab_server] => {
    "msg": "liveness: ok"
}
```

  On vérifie que les résultats sont bien affichés dans la console Ansible.

---

## Bonus task

### Print a custom message with only the dysfunctional(s) services in the Readiness check

**Input**
```bash
vagrant ssh gitlab_server
sudo gitlab-ctl stop redis
exit
```

**Output**
```text
ok: down: redis: 0s, normally up
```

  On stoppe `redis` pour simuler une panne avant de rejouer le playbook readiness.

**Input**
```bash
ansible-playbook /vagrant/playbooks/run.yml --tags readiness -i /tmp/vagrant-ansible/inventory/vagrant_ansible_local_inventory
```

**Output**
```text
TASK [gitlab : print readiness dysfunctional services] ***********************
ok: [gitlab_server] => {
    "msg": "readiness: dysfunctional services: redis"
}
```

  On n’affiche que les services en échec, en m’appuyant sur la réponse JSON (attribut `json`) du module `uri`.
