
# Tag 1

# Terraform
## Installation
### Repo einrichten (Debian)
```
sudo -i
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```
### Terraform installieren
```
apt update
apt install terraform
```

## Projekt initialisieren
Zuerst legen wir uns (als User tux!) ein Verzeichnis an:
```
cd
mkdir terraform
cd terraform
mkdir project1
cd project1
```
In diesem Verzeichnis legen wir uns die Konfiguration für Terraform an:
```
vi versions.tf
```
Inhalt der Datei:
```
terraform {
  required_providers {
    openstack = {
      source = "terraform-provider-openstack/openstack"
    }
  }
  required_version = ">= 0.13"
}
```
## Provider konfigurieren
Es wird eine neue Datei provider.tf im Workspace angelegt. Inhalt:
```
provider "openstack" {
  auth_url            = "https://fra1.citycloud.com:5000/v3.0"
  user_name           = "b1terraform-training"
  password            = "pCEkIrITPwEW38QVFz8n"
  region              = "Fra1"
  tenant_name         = "training-terraform-sandbox"
  project_domain_name = "CCP_Domain_38213"
  user_domain_name    = "CCP_Domain_38213"
}
```
## Eine erste Resource (VM):
neue Datei: server.tf

```
resource "openstack_compute_instance_v2" "msp-vm" {
  name        = "MSP Server"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = "1C-1GB-20GB"
  network {
    name = "net-to-external"
  }
}
```
### Erweiterung auf 3 gleiche VMs
```
resource "openstack_compute_instance_v2" "msp-vm" {
  name        = "MSP Server ${count.index + 1}"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = "1C-1GB-20GB"
  count       = 3
  network {
    name = "net-to-external"
  }
}
```
### Netzwerk und Subnet
neue Datei: network.tf
```
resource "openstack_networking_network_v2" "msp-net" {
  name           = "MSP Network"
  admin_state_up = true
}

resource "openstack_networking_subnet_v2" "msp-subnet" {
  name       = "MSP Subnet"
  ip_version = 4
  cidr       = "192.168.42.0/24"
  network_id = openstack_networking_network_v2.msp-net.id
}
```
Um das Netzwerk mit den Servern zu verbinden müssen wir die server.tf anpassen:
```
resource "openstack_compute_instance_v2" "msp-vm" {
  name        = "MSP Server ${count.index + 1}"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = "1C-1GB-20GB"
  count       = 3
  network {
    uuid = openstack_networking_network_v2.msp-net.id
  }
}
```
### Erweiterung um Router und Floatingip
```
resource "openstack_networking_network_v2" "msp-net" {
  name           = "MSP Network"
  admin_state_up = true
}

resource "openstack_networking_subnet_v2" "msp-subnet" {
  name       = "MSP Subnet"
  ip_version = 4
  cidr       = "192.168.42.0/24"
  network_id = openstack_networking_network_v2.msp-net.id
}

resource "openstack_networking_router_v2" "msp-router" {
  name                = "MSP Router"
  admin_state_up      = true
  external_network_id = data.openstack_networking_network_v2.external-net.id
}

resource "openstack_networking_router_interface_v2" "msp-interface" {
  router_id = openstack_networking_router_v2.msp-router.id
  subnet_id = openstack_networking_subnet_v2.msp-subnet.id
}

data "openstack_networking_network_v2" "external-net" {
  name = "ext-net"
}

resource "openstack_networking_floatingip_v2" "msp-float" {
  pool = "ext-net"
}

resource "openstack_compute_floatingip_associate_v2" "msp-fip-ass" {
  floating_ip = openstack_networking_floatingip_v2.msp-float.address
  instance_id = openstack_compute_instance_v2.msp-vm.0.id
}
```
### Keypair erzeugen
```
cd
ssh-keygen
```
Fragen einfach mit (enter) durchwinken
Public Key holen:
```
cat .ssh/id_rsa.pub
```
### Server mit keypair erweitern
server.tf:
```
resource "openstack_compute_instance_v2" "msp-vm" {
  name        = "MSP Server ${count.index + 1}"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = "1C-1GB-20GB"
  count       = 3
  key_pair    = "MSP"
  network {
    uuid = openstack_networking_network_v2.msp-net.id
  }
}

resource "openstack_compute_keypair_v2" "msp-key" {
  name       = "MSP"
  public_key = "(Das Ergebnis vom cat-Befehl OHNE tux@ansible...)"
}
```

# Tag 2
## Variablen
neue Datei: variables.tf
```
variable "flavor" {}

variable "vmcount" {}

variable "cidr" {}
```
neue Datei: terraform.tfvars
```
flavor  = "1C-1GB-20GB"
vmcount = 2
cidr    = "192.168.42.0/24"
```
Anpassung server.tf (Ausschnitt)
```
resource "openstack_compute_instance_v2" "msp-vm" {
  name        = "MSP Server ${count.index + 1}"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = var.flavor
  count       = var.vmcount
  key_pair    = "MSP"
  network {
    uuid = openstack_networking_network_v2.msp-net.id
  }
}
```
Anpassung network.tf (Ausschnitt)
```
resource "openstack_networking_subnet_v2" "msp-subnet" {
  name       = "MSP Subnet"
  ip_version = 4
  cidr       = var.cidr
  network_id = openstack_networking_network_v2.msp-net.id
}
```

## Outputs
neue Datei: outputs.tf
```
output "floating-IP" {
  value = openstack_networking_floatingip_v2.msp-float.address
}

output "server-ips" {
  value = openstack_compute_instance_v2.msp-vm.*.network.0.fixed_ip_v4
}
```
## Blockstorage
Ergänzung in der server.tf
```
resource "openstack_blockstorage_volume_v3" "msp-volume" {
  name  = "MSP Volume"
  size  = 3
  count = var.vmcount
}

resource "openstack_compute_volume_attach_v2" "msp-vol-att" {
  instance_id = openstack_compute_instance_v2.msp-vm[count.index].id
  volume_id   = openstack_blockstorage_volume_v3.msp-volume[count.index].id
  count       = var.vmcount
}
```
## Null Resource (remote und local-exec)
neue Datei: null.tf
```
resource "null_resource" "remote-command" {
  depends_on = [openstack_compute_floatingip_associate_v2.msp-fip-ass]

  provisioner "remote-exec" {
    inline = ["echo 'ich bin ein remote provisioner' > /tmp/provisioner.txt"]
    connection {
      type        = "ssh"
      user        = "debian"
      private_key = file("/home/tux/.ssh/id_rsa")
      #     private_key = file("/root/.ssh/id_rsa")   <== für diejenigen, die mit root arbeiten
      host = openstack_compute_floatingip_associate_v2.msp-fip-ass.floating_ip
    }
  }
}

resource "null_resource" "local-command" {
  depends_on = [openstack_compute_instance_v2.msp-vm]
  provisioner "local-exec" {
    command = "echo '${openstack_compute_instance_v2.msp-vm.0.access_ip_v4}' > ips.txt"
  }
}
```

## Module
### Vorbereitung:
Als der User, mit dem ihr arbeitet:
```
cd ~/terraform
mkdir workspace
mv project1 workspace
mkdir modules
cd modules
mkdir jumphost
cd jumphost
```
### Modul
main.tf:
```
variable "jumphost_flavor" {
}

variable "jumphost_port_id" {
}

resource "openstack_compute_instance_v2" "jumphost-msp-vm" {
  name        = "MSP Jumphost"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = var.jumphost_flavor
  key_pair    = "MSPjump"
  network {
    name = "net-to-external"
  }
  network {
    port = var.jumphost_port_id
  }
}

resource "openstack_compute_keypair_v2" "jumphost-msp-key" {
  name       = "MSPjump"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDPI5nnVAffaClCbkX0yhfHnB+tBg5jKMVwXTjLByzIk6HTtQPnocltv2+0HC3eFkbVs4zq/XeUS6riBXbT9HKmzz3853vh19K9G8wVRNxJZmvWNdZ1c/+pApcVOGm+/OBhivFg+U4727KDONqJgpD4X3QjfrLsK8w3dXLevzZxDNxJgiNzK7gOVIqunTINceIDybSjHgllhepeIGmHhf4qv6zoNfClJtmLWulYZh0NRiMApILq0sN7t30jFXHB9Kyxctumst7lVn0E9ovjpAXiiYeIkFyIZAcQf4OGlqbAj+5Octis4msDpc+wHhOiq+LAxPnxmsFfsiWL94CbeUgsKRUTjiY1sRyWTFD6YNln0yHaZzxUD3toMeq2Xft+8zKleb5IgS+kuDrL8HZnRvN5pI2CvcwNwTMydmcivXZg8U21z1cuBUReitL+JwzTU/OkDyUg9ZMBuvm7CfXLSH9T/Mwsrlsd5HvPuoSaD5dqEz7v4dnUyP79nw82gSFCofM="
}

resource "openstack_networking_floatingip_v2" "jumphost-msp-float" {
  pool = "ext-net"
}

resource "openstack_compute_floatingip_associate_v2" "jumphost-msp-fip-ass" {
  floating_ip = openstack_networking_floatingip_v2.jumphost-msp-float.address
  instance_id = openstack_compute_instance_v2.jumphost-msp-vm.id
}

output "jumphost-fip" {
  value = openstack_compute_floatingip_associate_v2.jumphost-msp-fip-ass.floating_ip
}

output "jumphost-subnet-id" {
  value = "e06323d7-ba2e-4afd-bf7d-adcb4fc2a809"
}
```

auch im Modul muss die Datei versions.tf enthalten sein

### Änderungen am Projekt:

network.tf:
```
resource "openstack_networking_network_v2" "msp-net" {
  name           = "MSP Network"
  admin_state_up = true
}

resource "openstack_networking_subnet_v2" "msp-subnet" {
  name       = "MSP Subnet"
  ip_version = 4
  cidr       = var.cidr
  network_id = openstack_networking_network_v2.msp-net.id
}

resource "openstack_networking_port_v2" "msp-port" {
  name           = "msp-port"
  network_id     = openstack_networking_network_v2.msp-net.id
  admin_state_up = "true"

  fixed_ip {
    subnet_id = openstack_networking_subnet_v2.msp-subnet.id
  }
}
```
null.tf:
```
resource "time_sleep" "wait_20_seconds" {
  depends_on = [module.jumphost]
  create_duration = "20s"
}

resource "null_resource" "remote-command" {
  depends_on = [time_sleep.wait_20_seconds]

  provisioner "remote-exec" {
    inline = ["echo 'ich bin ein remote provisioner' > /tmp/provisioner.txt"]
    connection {
      type        = "ssh"
      user        = "debian"
      private_key = file("/home/tux/.ssh/id_rsa")
      #     private_key = file("/root/.ssh/id_rsa")   <== für diejenigen, die mit root arbeiten
      host = module.jumphost.jumphost-fip
    }
  }
}

resource "null_resource" "local-command" {
  depends_on = [openstack_compute_instance_v2.msp-vm]
  provisioner "local-exec" {
    command = "echo '${openstack_compute_instance_v2.msp-vm.0.access_ip_v4}' > ips.txt"
  }
}
```
neue Datei: main.tf
```
module "jumphost" {
  source          = "../../modules/jumphost"
  jumphost_flavor = var.flavor
  jumphost_port_id = openstack_networking_port_v2.msp-port.id
}
```
# Tag 3 - Ansible

## Installation
```
sudo apt update
sudo apt install ansible-core
```
Überprüfung:
```
ansible --version
```
## Arbeitsverzeichnis einrichten
```
cd
mkdir ansible
cd ansible
```
## Inventory
neue Datei: inventory
```
debian1
debian2
ubuntu1
ubuntu2
redhat1
redhat2
suse1
suse2
```
## ssh einrichten
Falls noch kein ssh-key existiert (vorher als root gearbeitet) muss noch einer generiert werden:
```
ssh-keygen
```
Key auf die Server verteilen:
```
for i in ubuntu1 ubuntu2 suse1 suse2 redhat1 redhat2 debian1 debian2; do ssh-copy-id tux@$i; done
```
und dann abwechselnd yes (hostkey) eintippen und das Passwort für den Benutzer Tux (b1s)
### Test der Verbindung
```
ansible -i inventory all -m ping
```
jetzt muss alles grün sein (bis auf Warnings, die wir ignorieren ;) )

## Inventory mit Gruppen
```
debian1
debian2
ubuntu1
ubuntu2
redhat1
redhat2
suse1
suse2

[debian]
debian1
debian2

[ubuntu]
ubuntu1
ubuntu2

[redhat]
redhat1
redhat2

[suse]
suse1
suse2
```

## sudo im ansible Kontext
Ansible braucht für viele Aktionen sudo-Rechte. Dazu gibt es bei einem ad-hoc Kommando den Schalter ```--become``` oder in Kurzform ```-b```

### Lösung Folie 145
Aufage 1: wird nicht gemacht (deprecated)
Aufgabe 2:
```
ansible -i inventory ubuntu,suse1 -m ansible.builtin.user -a "name=markus shell=/bin/bash" -b
```
Aufgabe 3:
```
ansible redhat -i inventory -m ansible.builtin.setup -a "filter=ansible_eth0"
```

## Collection installieren und auflisten
```
ansible-galaxy collection install ansible.posix
```
```
ansible-galaxy collection list
```
## Playbooks
### das erste Playbook
```
---
- name: First playbook
  hosts: all
  become: true
  tasks:
   - name: Ensure that the directory /home/tux/foo is present
      ansible.builtin.file:
        path: /home/tux/foo
        owner: tux
        mode: "0755"
        state: directory 
```
Aufruf des playbooks:
```
ansible-playbook -i inventory playbook1.yml
```
## Exkurs: yamllint
```
sudo apt install yamllint
```
## Konfiguration von Ansible
```
cd ~/ansible
ansible-config init --disabled > ansible.cfg
```
Jetzt kann man dann mit einem Editor die Datei anpassen und damit ansible konfigurieren.
In Zeile 137 setzen wir:
```
inventory=/home/tux/ansible/inventory
```
## Lösung Folie 182
```
---
- name: ensure some software
  become: true
  hosts: all
  tasks:
    - name: ensure git present on ubuntu1 and suse2
      ansible.builtin.package:
        name: git
        state: present
      when: ansible_hostname == "ubuntu1" or ansible_hostname == "suse2"
    - name: ensure package curl on non Suse ans non Debian family hosts
      ansible.builtin.package:
        name: curl
        state: present
      when: ansible_os_family != "Suse" and ansible_os_family != "Debian"
```

# Tag 4
## Übung zu loop und registered variable
```
---
- name: example for registered variable and a loop
  hosts: suse
  tasks:
    - name: create count variable
      ansible.builtin.command:
        cmd: "seq 1 20"
      register: counter
    - name: show counter variable
      ansible.builtin.debug:
        var: "counter"
    - name: create dirs in /tmp
      ansible.builtin.file:
        path: "/tmp/dir{{ item }}"
        state: directory
        owner: tux
        mode: "0755"
      loop: "{{ counter.stdout_lines }}"
```
## ansible-Verzeichnis aufräumen und Standard Ornderstruktur anlegen
```
cd ~/ansible
mkdir playbooks
mkdir roles
mv *.yml playbooks
```
ggfs noch überflüssige Dateien wegräumen ;)

## Playbook zum Erzeugen einer Rollenstruktur (customized) statt ansible-galaxy init
```
---
- name: Standard playbook to create customized role structure
  hosts: localhost
  vars:
    roledir: "/home/tux/ansible/roles"
    rolename: "default"
    directories:
      - tasks
      - vars
      - defaults
      - files
      - templates
      - meta
      - handlers
  tasks:
    - name: create roledirs
      ansible.builtin.file:
        path: "{{ roledir }}/{{ rolename }}/{{ item }}"
        state: directory
        owner: tux
        mode: "0755"
      loop: "{{ directories }}"
    - name: create main.yml where needed
      ansible.builtin.file:
        path: "{{ roledir }}/{{ rolename }}/{{ item }}/main.yml"
        state: touch
        owner: tux
        mode: "0644"
      loop: "{{ directories }}"
      when: item in ['tasks', 'vars', 'defaults', 'meta', 'handlers']
    - name: fill main.yml with my standards
      ansible.builtin.lineinfile:
        path: "{{ roledir }}/{{ rolename }}/{{ item }}/main.yml"
        state: present
        line: "---"
      loop: "{{ directories }}"
      when: item in ['tasks', 'vars', 'defaults', 'meta', 'handlers'] 
```
Aufruf dann wie folgt:
```
ansible-playbook playbooks/create_role.yml -e rolename=apache -e roledir=/home/tux/test/roles
```

# Tag 5: Projekt TLRZ
auf dem ansible-controller
```
cd
mkdir tlrz-project
cd tlrz-project
cp ../terraform/workspace/project1/versions.tf .
terraform init
source ~/openrc.sh
```
## Terraform
server.tf:
```
resource "openstack_compute_instance_v2" "frontend" {
  name        = "MSP Webserver Frontend"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = "1C-1GB-20GB"
  key_pair    = "MSP"
  network {
    name = "net-to-external"
  }
}

resource "openstack_compute_instance_v2" "backend" {
  name        = "MSP Database Backend"
  image_name  = "Debian 12 Bookworm x86_64"
  flavor_name = "2C-2GB-20GB"
  key_pair    = "MSP"
  network {
    name = "net-to-external"
  }
}

resource "openstack_compute_keypair_v2" "msp-key" {
  name       = "MSP"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDPI5nnVAffaClCbkX0yhfHnB+tBg5jKMVwXTjLByzIk6HTtQPnocltv2+0HC3eFkbVs4zq/XeUS6riBXbT9HKmzz3853vh19K9G8wVRNxJZmvWNdZ1c/+pApcVOGm+/OBhivFg+U4727KDONqJgpD4X3QjfrLsK8w3dXLevzZxDNxJgiNzK7gOVIqunTINceIDybSjHgllhepeIGmHhf4qv6zoNfClJtmLWulYZh0NRiMApILq0sN7t30jFXHB9Kyxctumst7lVn0E9ovjpAXiiYeIkFyIZAcQf4OGlqbAj+5Octis4msDpc+wHhOiq+LAxPnxmsFfsiWL94CbeUgsKRUTjiY1sRyWTFD6YNln0yHaZzxUD3toMeq2Xft+8zKleb5IgS+kuDrL8HZnRvN5pI2CvcwNwTMydmcivXZg8U21z1cuBUReitL+JwzTU/OkDyUg9ZMBuvm7CfXLSH9T/Mwsrlsd5HvPuoSaD5dqEz7v4dnUyP79nw82gSFCofM="
}

resource "openstack_blockstorage_volume_v3" "database-vol" {
  name = "MSP database disk"
  size = 20
}

resource "openstack_compute_volume_attach_v2" "database_att" {
  instance_id = openstack_compute_instance_v2.backend.id
  volume_id   = openstack_blockstorage_volume_v3.database-vol.id
}
```
network.tf
```
resource "openstack_networking_floatingip_v2" "fip_front" {
  pool = "ext-net"
}

resource "openstack_compute_floatingip_associate_v2" "fip_front_ass" {
  instance_id = openstack_compute_instance_v2.frontend.id
  floating_ip = openstack_networking_floatingip_v2.fip_front.address
}

resource "openstack_networking_floatingip_v2" "fip_back" {
  pool = "ext-net"
}

resource "openstack_compute_floatingip_associate_v2" "fip_back_ass" {
  instance_id = openstack_compute_instance_v2.backend.id
  floating_ip = openstack_networking_floatingip_v2.fip_back.address
}
```
null.tf
```
resource "null_resource" "backend" {
  depends_on = [openstack_compute_floatingip_associate_v2.fip_back_ass,
  openstack_compute_instance_v2.backend]
  # wait for connectivity
  provisioner "remote-exec" {
    inline = ["sudo cloud-init status --wait"]
    connection {
      type        = "ssh"
      user        = "debian"
      host        = openstack_compute_floatingip_associate_v2.fip_back_ass.floating_ip
      private_key = file("/home/tux/.ssh/id_rsa")
    }
  }

  provisioner "local-exec" {
    command = "ansible-playbook -u 'debian' -i '${openstack_compute_floatingip_associate_v2.fip_back_ass.floating_ip},' --ssh-common-args='-o StrictHostKeyChecking=no' main.yml -t backend -t common"
  }
}

resource "null_resource" "frontend" {
  depends_on = [openstack_compute_floatingip_associate_v2.fip_front_ass,
  openstack_compute_instance_v2.backend]

  # wait for connectivity
  provisioner "remote-exec" {
    inline = ["sudo cloud-init status --wait"]
    connection {
      type        = "ssh"
      user        = "debian"
      host        = openstack_compute_floatingip_associate_v2.fip_front_ass.floating_ip
      private_key = file("/home/tux/.ssh/id_rsa")
    }
  }

  provisioner "local-exec" {
    command = "ansible-playbook -u 'debian' -i '${openstack_compute_floatingip_associate_v2.fip_front_ass.floating_ip},' --ssh-common-args='-o StrictHostKeyChecking=no' main.yml -t frontend -t common"
  }
}
```
## Ansible
im Projektordner:
```
mkdir roles
cd roles
ansible-galaxy init common
ansible-galaxy init database
ansible-galaxy init webserver
```
Playbook main.yml
```
---
- hosts: all
  become: true
  roles:
    - common
    - database
    - webserver
```
### Rolle common
roles/common/tasks/main.yml
```
---
- name: common tasks for all servers
  tags: common
  block:
    - name: wait for connection
      ansible.builtin.ping: {}
      register: pingresult
      until: pingresult['failed'] == false
      retries: 10
      delay: 5

    - name: create service user
      ansible.builtin.user:
        name: service
        generate_ssh_key: true
        home: /opt/service
        shell: /bin/bash
        system: true

    - name: install standard software packages
      ansible.builtin.apt:
        update_cache: true
        name:
          - curl
          - htop
          - cowsay
          - fortune
          - nano
        state: latest
```
### Rolle database
roles/database/tasks/main.yml
```
---
- name: create the database backend
  tags: backend
  block:
    - name: install mariadb
      ansible.builtin.apt:
        update_cache: true
        name: mariadb-server
        state: present
    - name: configure mariadb
      ansible.builtin.template:
        src: 50-server.cnf.j2
        dest: /etc/mysql/mariadb.conf.d/50-server.cnf
        owner: root
        group: root
        mode: "0644"
      notify: restart mariadb
```
roles/database/handlers/main.yml
```
---
- name: restart mariadb
  ansible.builtin.service:
    name: mariadb
    state: restarted
```
roles/database/vars/main.yml
```
---
db_local_only: false
backend_ip: 0.0.0.0
```
roles/database/templates/50-server.cnf.j2
```
[server]
[mysqld]
pid-file                = /run/mysqld/mysqld.pid
basedir                 = /usr
{% if db_local_only %} 
bind-address            = 127.0.0.1
{% else %}
bind-address            = {{ backend_ip }}
{% endif %}
expire_logs_days        = 10
character-set-server  = utf8mb4
collation-server      = utf8mb4_general_ci
[embedded]
[mariadb]
[mariadb-10.11]
```
### Rolle webserver
roles/webserver/tasks/main.yml
```
---
- name: create the frontend
  tags: frontend
  block:
    - name: install nginx
      ansible.builtin.apt:
        update_cache: true
        name: nginx
        state: present
    - name: configure nginx
      ansible.builtin.copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
      notify: restart nginx
    - name: remove default vhost
      ansible.builtin.file:
        path: /etc/nginx/{{ item }}/default
        state: absent
      loop:
        - sites-enabled
        - sites-available
      notify: restart nginx
    - name: configure vhost
      ansible.builtin.template:
        src: tlrz.de.j2
        dest: /etc/nginx/sites-available/{{ servername }}
        owner: root
        group: root
        mode: "0644"
      notify: restart nginx
    - name: enable the site
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/{{ servername }}
        src: /etc/nginx/sites-available/{{ servername }}
        state: link
        owner: root
        group: root
        mode: "0644"
      notify: restart nginx
```
roles/webserver/handlers/main.yml
```
---
- name: restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```
roles/webserver/files/nginx.conf
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;
events {
	worker_connections 512;
}
http {
	sendfile on;
	tcp_nopush on;
	types_hash_max_size 2048;
	include /etc/nginx/mime.types;
	default_type application/octet-stream;
	ssl_prefer_server_ciphers on;
	access_log /var/log/nginx/access.log;
	gzip on;
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}
```
roles/webserver/templates/tlrz.de.j2
```
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	root /var/www/html;
	index index.html;
	server_name {{ servername }};
	location / {
		try_files $uri $uri/ =404;
	}
}
```
roles/webserver/vars/main.yml
```
---
servername: tlrz.de
```
