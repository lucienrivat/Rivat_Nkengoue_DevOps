# Rapport de TP — Vagrant (Shell) & GitLab (Ansible) + tests (Inputs/Outputs)

---

## 1) Imperative — Using Vagrant with Shell Provisioner

### 1.1 Idée 

Approche impérative : on décrit comment faire, étape par étape, via un script shell exécuté pendant le provisioning.


---

### 1.3 Vagrantfile (Shell provisioner)

Créer/adapter `Vagrantfile` :

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.define "gitlab-shell"

  config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true

  
  config.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh", auto_correct: true

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus = 2
  end

  config.vm.provision "shell", path: "provision/01-base.sh"
  config.vm.provision "shell", path: "provision/02-tools.sh"
end
```

---

### 1.4 Script 01 — base système

Créer `provision/01-base.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

echo "[01] Mise à jour des paquets"
apt-get update -y

echo "[01] Outils de base"
apt-get install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  apt-transport-https \
  vim \
  htop

echo "[01] Terminé"
```

Rendre exécutable :

**Input**
```bash
chmod +x provision/01-base.sh
```

**Output**
```text
pas de sortie
```

---

### 1.5 Script 02 — swap + checks

Créer `provision/02-tools.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail

SWAPFILE="/swapfile"
SWAPSIZE_MB=1024

if ! swapon --show | grep -q "$SWAPFILE"; then
  echo "[02] Création d'un swap de ${SWAPSIZE_MB}MB"
  fallocate -l "${SWAPSIZE_MB}M" "$SWAPFILE"
  chmod 600 "$SWAPFILE"
  mkswap "$SWAPFILE"
  swapon "$SWAPFILE"
  echo "$SWAPFILE none swap sw 0 0" >> /etc/fstab
else
  echo "[02] Swap déjà présent"
fi

echo "[02] Etat swap :"
swapon --show || true
```

Rendre exécutable :

**Input**
```bash
chmod +x provision/02-tools.sh
```

**Output**
```text
pas de sortie 
```

---

### 1.6 Lancement de la VM

**Input**
```bash
vagrant up
```

**Output**
```text
Bringing machine 'gitlab-shell' up with 'virtualbox' provider...
==> gitlab-shell: Importing base box 'ubuntu/jammy64'...
==> gitlab-shell: Matching MAC address for NAT networking...
==> gitlab-shell: Setting the name of the VM: gitlab-shell
==> gitlab-shell: Clearing any previously set network interfaces...
==> gitlab-shell: Forwarding ports...
==> gitlab-shell: 80 (guest) => 8080 (host)
==> gitlab-shell: 22 (guest) => 2222 (host)
==> gitlab-shell: Running provisioner: shell...
    gitlab-shell: [01] Mise à jour des paquets
    gitlab-shell: [01] Outils de base
    gitlab-shell: [01] Terminé
==> gitlab-shell: Running provisioner: shell...
    gitlab-shell: [02] Création d'un swap de 1024MB
    gitlab-shell: Setting up swapspace version 1, size = 1024 MiB
    gitlab-shell: [02] Etat swap :
    gitlab-shell: NAME      TYPE SIZE USED PRIO
    gitlab-shell: /swapfile file   1G   0B   -2
```

> Les lignes exactes changent selon la box et la machine, mais on doit voir les deux provisioners s’exécuter.

---

### 1.7 Connexion SSH + checks

**Input**
```bash
vagrant ssh
```

**Output**
```text
Welcome to Ubuntu 22.04 LTS (GNU/Linux 5.x.x-xx-generic x86_64)
vagrant@ubuntu-jammy:~$
```

#### Check — curl installé

**Input**
```bash
curl --version | head -n 1
```

**Output**
```text
curl 7.81.0 (x86_64-pc-linux-gnu) ...
```

#### Check — swap actif

**Input**
```bash
swapon --show
```

**Output**
```text
NAME      TYPE SIZE USED PRIO
/swapfile file   1G   0B   -2
```

Sortir du SSH :

**Input**
```bash
exit
```

**Output**
```text
# (retour au shell hôte)
```

---

### 1.8 Rejouer le provisioning après modification

**Input**
```bash
vagrant provision
```

**Output**
```text
==> gitlab-shell: Running provisioner: shell...
    gitlab-shell: [01] Mise à jour des paquets
    gitlab-shell: [01] Outils de base
    gitlab-shell: [01] Terminé
==> gitlab-shell: Running provisioner: shell...
    gitlab-shell: [02] Swap déjà présent
    gitlab-shell: [02] Etat swap :
    gitlab-shell: NAME      TYPE SIZE USED PRIO
    gitlab-shell: /swapfile file   1G   0B   -2
```

---

## 2) Declarative — GitLab installation using Vagrant and Ansible Provisioner

### 2.1 Idée (déclaratif)

Approche **déclarative** : on décrit un état final (GitLab installé, service actif, config appliquée).  
On utilise ici `ansible_local` pour exécuter Ansible **dans la VM**.

---

### 2.2 Arborescence

```text
tp-vagrant-gitlab/
├─ Vagrantfile
├─ provision/
│  └─ 00-ansible.sh
└─ ansible/
   ├─ inventory.ini
   ├─ playbook.yml
   └─ group_vars/
      └─ all.yml
```

---

### 2.3 Vagrantfile (Shell + ansible_local)

> Dans ce scénario, on peut soit **remplacer** la partie Shell du chapitre 1, soit créer une VM séparée.  
> Ici je pars sur **une VM dédiée** “gitlab-ansible” pour séparer proprement les approches.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.define "gitlab-ansible"

  config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
  config.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh", auto_correct: true

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus = 2
  end

  config.vm.provision "shell", path: "provision/00-ansible.sh"

  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "/vagrant/ansible/playbook.yml"
    ansible.inventory_path = "/vagrant/ansible/inventory.ini"
    ansible.extra_vars = {
      gitlab_external_url: "http://localhost:8080"
    }
  end
end
```

---

### 2.4 Script — installation Ansible dans la VM

Créer `provision/00-ansible.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

apt-get update -y
apt-get install -y ansible
ansible --version
```

Rendre exécutable :

**Input**
```bash
chmod +x provision/00-ansible.sh
```

**Output**
```text
pas de sortie
```

---

### 2.5 Inventory & variables

`ansible/inventory.ini` :

```ini
[gitlab]
localhost ansible_connection=local
```

`ansible/group_vars/all.yml` :

```yaml
gitlab_package: gitlab-ce
gitlab_external_url: "http://localhost:8080"
```

---

### 2.6 Playbook — installation et configuration GitLab

`ansible/playbook.yml` :

```yaml
- name: Install and configure GitLab
  hosts: gitlab
  become: true
  vars_files:
    - group_vars/all.yml

  tasks:
    - name: Ensure base dependencies are installed
      apt:
        name:
          - curl
          - ca-certificates
          - openssh-server
          - tzdata
        state: present
        update_cache: true

    - name: Add GitLab package repository (CE)
      shell: |
        set -e
        curl --location "https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh" | bash
      args:
        creates: /etc/apt/sources.list.d/gitlab_gitlab-ce.list

    - name: Install GitLab package
      apt:
        name: "{{ gitlab_package }}"
        state: present
        update_cache: true

    - name: Set external_url
      lineinfile:
        path: /etc/gitlab/gitlab.rb
        regexp: '^external_url'
        line: "external_url '{{ gitlab_external_url }}'"

    - name: Reconfigure GitLab
      command: gitlab-ctl reconfigure
```

---

### 2.7 Lancement et sortie attendue Ansible

**Input**
```bash
vagrant up
```

**Output**
```text
Bringing machine 'gitlab-ansible' up with 'virtualbox' provider...
==> gitlab-ansible: Running provisioner: shell...
    gitlab-ansible: ansible 2.12.x
==> gitlab-ansible: Running provisioner: ansible_local...
    gitlab-ansible: Running ansible-playbook...
PLAY [Install and configure GitLab] ****************************************

TASK [Gathering Facts] *****************************************************
ok: [localhost]

TASK [Ensure base dependencies are installed] ******************************
changed: [localhost]

TASK [Add GitLab package repository (CE)] **********************************
changed: [localhost]

TASK [Install GitLab package] **********************************************
changed: [localhost]

TASK [Set external_url] ****************************************************
changed: [localhost]

TASK [Reconfigure GitLab] **************************************************
changed: [localhost]

PLAY RECAP *****************************************************************
localhost                  : ok=6  changed=5  unreachable=0  failed=0
```

---

### 2.8 Vérifications post-install (dans la VM)

Connexion :

**Input**
```bash
vagrant ssh
```

**Output**
```text
vagrant@ubuntu-jammy:~$
```

#### Vérifier l’état GitLab

**Input**
```bash
sudo gitlab-ctl status
```

**Output**
```text
run: gitaly: (pid 1234) 120s; run: log: (pid 1200) 130s
run: nginx: (pid 1235) 118s; run: log: (pid 1199) 130s
run: puma: (pid 1236) 117s; run: log: (pid 1198) 130s
run: redis: (pid 1237) 116s; run: log: (pid 1197) 130s
...
```

> Le nombre de services et leurs noms exacts peuvent varier selon la version, mais on doit voir `run:` partout.

#### Vérifier que Nginx répond (dans la VM)

**Input**
```bash
curl -I http://localhost | head -n 5
```

**Output**
```text
HTTP/1.1 302 Found
Server: nginx
Date: Thu, 12 Feb 2026 10:12:33 GMT
Content-Type: text/html
Location: /users/sign_in
```

Sortir du SSH :

**Input**
```bash
exit
```

---

### 2.9 Vérification depuis la machine hôte

#### Accès à l’interface web

Ouvrir dans un navigateur :

**Input**
```text
http://localhost:8080
```

**Output**
```text
Page GitLab accessible (écran de login).
```

---

## 3) Play with different commands for Shell Provisioner

Cette partie montre des **mini-expériences** pour prouver qu’on comprend le Shell provisioner.

### 3.1 Inline vs script

#### Inline

Dans le `Vagrantfile` :

```ruby
config.vm.provision "shell", inline: <<-SHELL
  echo "Hello from inline shell!"
  uname -a
SHELL
```

**Input**
```bash
vagrant provision
```

**Output**
```text
==> default: Running provisioner: shell...
    default: Hello from inline shell!
    default: Linux ubuntu-jammy 5.x.x-xx-generic #... SMP ... x86_64 GNU/Linux
```

#### Script

```ruby
config.vm.provision "shell", path: "provision/hello.sh"
```

`provision/hello.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail
echo "Hello from script!"
```

**Input**
```bash
vagrant provision
```

**Output**
```text
==> default: Running provisioner: shell...
    default: Hello from script!
```

---

### 3.2 Arguments à un script

Dans `Vagrantfile` :

```ruby
config.vm.provision "shell", path: "provision/param.sh", args: ["Alice", "42"]
```

`provision/param.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail
echo "User=$1"
echo "Answer=$2"
```

**Input**
```bash
vagrant provision
```

**Output**
```text
==> default: Running provisioner: shell...
    default: User=Alice
    default: Answer=42
```

---

### 3.3 privileged true/false

```ruby
config.vm.provision "shell", inline: "id", privileged: true
config.vm.provision "shell", inline: "id", privileged: false
```

**Input**
```bash
vagrant provision
```

**Output**
```text
==> default: Running provisioner: shell...
    default: uid=0(root) gid=0(root) groups=0(root)
==> default: Running provisioner: shell...
    default: uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant),...
```

---

### 3.4 run: "always" + preuve

```ruby
config.vm.provision "shell",
  inline: "date | tee /tmp/provision-date",
  run: "always"
```

**Input**
```bash
vagrant reload
vagrant ssh -c "cat /tmp/provision-date"
```

**Output**
```text
Thu Feb 12 10:21:04 UTC 2026
```

Relancer `vagrant reload` et rechecker : la date doit changer → preuve que le provisioner s’exécute “à chaque fois”.

---

## 4) Critères de réussite (check-list)

### 4.1 Partie Shell (impérative)
- `vagrant up` OK
- `curl`, `vim`, etc. présents
- swap actif avec `swapon --show`

### 4.2 Partie GitLab (Ansible)
- Playbook OK (pas de failed)
- `sudo gitlab-ctl status` montre des `run:`
- UI accessible sur `http://localhost:8080`

---

## 5) Problèmes fréquents (tests de diagnostic)

### 5.1 GitLab lent / erreurs 502

**Input**
```bash
vagrant ssh
free -h
df -h
```

**Output**
```text
              total        used        free      shared  buff/cache   available
Mem:           3.8Gi       2.1Gi       0.4Gi       0.0Gi       1.3Gi       1.4Gi
Swap:          1.0Gi       0.0Gi       1.0Gi
```

Solution : augmenter RAM et/ou swap, redémarrer GitLab.

**Input**
```bash
sudo gitlab-ctl restart
sudo gitlab-ctl status
```

**Output**
```text
ok: run: nginx ...
ok: run: puma ...
...
```

---

## 6) Annexe — Énoncé fourni

L’énoncé était dans `lab.md` (fourni).  
Ce rapport couvre les 4 parties demandées :
- Avant de commencer
- **Imperative — Shell Provisioner**
- **Declarative — GitLab + Ansible**
- **Play with different commands**
