# Kritik Dosyaların Tam İçeriği

Bu belge mevcut proje dosyalarından üretildi. Çalıştırılan kaynaklar yanlarında belirtilen gerçek dosyalardır.

## `ansible.cfg`

```ini
[defaults]
inventory = inventory/staging/hosts.ini
roles_path = roles
collections_path = collections
forks = 20
host_key_checking = True
retry_files_enabled = False
interpreter_python = auto_silent
timeout = 30

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
control_path_dir = ~/.ansible/cp
```

## `requirements.yml`

```yaml
collections:
  - name: community.general
    version: '>=10.0.0,<12.0.0'
  - name: ansible.posix
    version: '>=1.6.0,<3.0.0'
```

## `site.yml`

```yaml
---
- name: Provision Ubuntu servers
  hosts: servers
  become: true
  gather_facts: true
  pre_tasks:
    - name: Require a supported Ubuntu release
      ansible.builtin.assert:
        that:
          - ansible_distribution == 'Ubuntu'
          - ansible_distribution_version in ['22.04', '24.04']
        fail_msg: This project supports Ubuntu 22.04 and 24.04 only.

    - name: Require an administrator with a real SSH key before hardening
      ansible.builtin.assert:
        that:
          - admin_users | length > 0
          - admin_users | selectattr('name', 'defined') | list | length == admin_users | length
          - admin_users | selectattr('ssh_keys', 'defined') | list | length == admin_users | length
          - (admin_users | map(attribute='ssh_keys') | map('length') | min | default(0)) | int > 0
        fail_msg: Set admin_users with real SSH public keys in the environment vars before running.

    - name: Require usable sudo credentials for every administrator
      ansible.builtin.assert:
        that:
          - (item.passwordless_sudo | default(false) | bool) or (item.password_hash is defined and (item.password_hash | length > 0))
        fail_msg: Each administrator needs passwordless_sudo true or a known password_hash for sudo.
      loop: "{{ admin_users }}"
      no_log: true

  roles:
    - role: common
    - role: users
    - role: security
    - role: docker
```

## `group_vars/all.yml`

```yaml
---
common_timezone: Etc/UTC
common_packages:
  - ca-certificates
  - curl
  - git
  - htop
  - jq
  - unzip
  - vim
  - wget

security_ssh_port: 22
security_allowed_tcp_ports: []
security_allowed_udp_ports: []
security_ssh_allow_users: []

docker_daemon_log_max_size: 10m
docker_daemon_log_max_file: '3'
docker_daemon_live_restore: true
```

## `inventory/production/group_vars/all.yml`

```yaml
---
# Replace the reserved sample IP in hosts.ini and add real public keys below.
admin_users: []
```

## `inventory/production/group_vars/vault.example.yml`

```yaml
---
# Copy to vault.yml and encrypt with ansible-vault encrypt.
vault_admin_password_hash: '$6$REPLACE_WITH_REAL_SHA512_HASH'
```

## `inventory/production/hosts.ini`

```ini
[servers]
production-01 ansible_host=192.0.2.20 ansible_user=ubuntu
```

## `inventory/staging/group_vars/all.yml`

```yaml
---
# Replace the reserved sample IP in hosts.ini and add real public keys below.
admin_users: []
```

## `inventory/staging/group_vars/vault.example.yml`

```yaml
---
# Copy to vault.yml and encrypt with ansible-vault encrypt.
# Generate a SHA-512 hash with ansible-vault encrypt_string or mkpasswd -m sha-512.
vault_admin_password_hash: '$6$REPLACE_WITH_REAL_SHA512_HASH'
```

## `inventory/staging/hosts.ini`

```ini
[servers]
staging-01 ansible_host=192.0.2.10 ansible_user=ubuntu
```

## `roles/common/defaults/main.yml`

```yaml
---
common_timezone: Etc/UTC
common_packages:
  - ca-certificates
  - curl
  - git
  - htop
common_upgrade_packages: true
```

## `roles/common/handlers/main.yml`

```yaml
---
[]
```

## `roles/common/meta/main.yml`

```yaml
---
galaxy_info:
  role_name: common
  author: project
  description: Ubuntu base packages, timezone, and Chrony
  license: MIT
  min_ansible_version: '2.16'
  platforms:
    - name: Ubuntu
      versions: [jammy, noble]
dependencies: []
```

## `roles/common/tasks/main.yml`

```yaml
---
- name: Refresh APT cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600

- name: Upgrade installed packages
  ansible.builtin.apt:
    upgrade: safe
  when: common_upgrade_packages | bool

- name: Install base tools and Chrony
  ansible.builtin.apt:
    name: "{{ common_packages + ['chrony', 'tzdata'] }}"
    state: present

- name: Set timezone
  community.general.timezone:
    name: "{{ common_timezone }}"

- name: Enable and start Chrony
  ansible.builtin.service:
    name: chrony
    enabled: true
    state: started
```

## `roles/docker/defaults/main.yml`

```yaml
---
docker_daemon_log_max_size: 10m
docker_daemon_log_max_file: '3'
docker_daemon_live_restore: true
```

## `roles/docker/handlers/main.yml`

```yaml
---
- name: Restart Docker
  ansible.builtin.service:
    name: docker
    state: restarted
```

## `roles/docker/meta/main.yml`

```yaml
---
galaxy_info:
  role_name: docker
  author: project
  description: Docker Engine and Compose from the official repository
  license: MIT
  min_ansible_version: '2.16'
  platforms:
    - name: Ubuntu
      versions: [jammy, noble]
dependencies: []
```

## `roles/docker/tasks/main.yml`

```yaml
---
- name: Install prerequisites for Docker APT repository
  ansible.builtin.apt:
    name:
      - ca-certificates
      - python3-debian
    state: present

- name: Create APT keyring directory
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Download official Docker signing key
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    owner: root
    group: root
    mode: '0644'

- name: Configure official Docker APT repository
  ansible.builtin.deb822_repository:
    name: docker
    types: deb
    uris: https://download.docker.com/linux/ubuntu
    suites: "{{ ansible_distribution_release }}"
    components: stable
    architectures: "{{ 'amd64' if ansible_architecture == 'x86_64' else 'arm64' if ansible_architecture == 'aarch64' else ansible_architecture }}"
    signed_by: /etc/apt/keyrings/docker.asc
    state: present
  register: docker_repository

- name: Refresh APT cache after adding Docker repository
  ansible.builtin.apt:
    update_cache: true
  when: docker_repository.changed

- name: Install Docker Engine and Compose plugin
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present

- name: Create Docker configuration directory
  ansible.builtin.file:
    path: /etc/docker
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Configure Docker daemon logging and live restore
  ansible.builtin.template:
    src: daemon.json.j2
    dest: /etc/docker/daemon.json
    owner: root
    group: root
    mode: '0644'
  notify: Restart Docker

- name: Enable and start Docker
  ansible.builtin.service:
    name: docker
    enabled: true
    state: started
```

## `roles/docker/templates/daemon.json.j2`

```jinja
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "{{ docker_daemon_log_max_size }}",
    "max-file": "{{ docker_daemon_log_max_file }}"
  },
  "live-restore": {{ docker_daemon_live_restore | bool | lower }}
}
```

## `roles/security/defaults/main.yml`

```yaml
---
security_ssh_port: 22
security_ssh_previous_port: 22
security_allowed_tcp_ports: []
security_allowed_udp_ports: []
security_ssh_allow_users: []
security_fail2ban_bantime: 1h
security_fail2ban_findtime: 10m
security_fail2ban_maxretry: 5
```

## `roles/security/handlers/main.yml`

```yaml
---
- name: Validate SSH configuration
  ansible.builtin.command: /usr/sbin/sshd -t
  changed_when: false
  listen: Validate and restart SSH

- name: Restart SSH
  ansible.builtin.service:
    name: ssh
    state: restarted
  listen: Validate and restart SSH

- name: Restart Fail2ban
  ansible.builtin.service:
    name: fail2ban
    state: restarted
```

## `roles/security/meta/main.yml`

```yaml
---
galaxy_info:
  role_name: security
  author: project
  description: UFW, OpenSSH hardening, and Fail2ban
  license: MIT
  min_ansible_version: '2.16'
  platforms:
    - name: Ubuntu
      versions: [jammy, noble]
dependencies: []
```

## `roles/security/tasks/main.yml`

```yaml
---
- name: Validate firewall and SSH settings
  ansible.builtin.assert:
    that:
      - security_ssh_port | int >= 1
      - security_ssh_port | int <= 65535
      - security_ssh_port | int not in (security_allowed_tcp_ports | map('int') | list)
    fail_msg: SSH port must be valid and must not be duplicated in security_allowed_tcp_ports.

- name: Install UFW, Fail2ban and OpenSSH server
  ansible.builtin.apt:
    name:
      - ufw
      - fail2ban
      - openssh-server
    state: present

- name: Set default incoming firewall policy
  community.general.ufw:
    direction: incoming
    policy: deny

- name: Set default outgoing firewall policy
  community.general.ufw:
    direction: outgoing
    policy: allow

- name: Allow SSH before enabling the firewall
  community.general.ufw:
    rule: allow
    port: "{{ security_ssh_port | string }}"
    proto: tcp

- name: Keep the current SSH connection port open during migration
  community.general.ufw:
    rule: allow
    port: "{{ ansible_port | default(22) | string }}"
    proto: tcp
  when: (ansible_port | default(22) | int) != (security_ssh_port | int)

- name: Close the previous SSH port after reconnecting on the new port
  community.general.ufw:
    rule: allow
    port: "{{ security_ssh_previous_port | string }}"
    proto: tcp
    delete: true
  when:
    - (security_ssh_previous_port | int) != (security_ssh_port | int)
    - (ansible_port | default(22) | int) == (security_ssh_port | int)

- name: Allow required TCP ports
  community.general.ufw:
    rule: allow
    port: "{{ item | string }}"
    proto: tcp
  loop: "{{ security_allowed_tcp_ports }}"

- name: Allow required UDP ports
  community.general.ufw:
    rule: allow
    port: "{{ item | string }}"
    proto: udp
  loop: "{{ security_allowed_udp_ports }}"

- name: Enable UFW
  community.general.ufw:
    state: enabled

- name: Disable Ubuntu 24.04 SSH socket activation
  ansible.builtin.systemd_service:
    name: ssh.socket
    enabled: false
    state: stopped
  when: ansible_distribution_version == '24.04'

- name: Enable the persistent SSH service
  ansible.builtin.service:
    name: ssh
    enabled: true
    state: started

- name: Install SSH hardening drop-in
  ansible.builtin.template:
    src: 00-ansible-hardening.conf.j2
    dest: /etc/ssh/sshd_config.d/00-ansible-hardening.conf
    owner: root
    group: root
    mode: '0644'
  notify: Validate and restart SSH

- name: Install Fail2ban SSH jail
  ansible.builtin.template:
    src: sshd.local.j2
    dest: /etc/fail2ban/jail.d/sshd.local
    owner: root
    group: root
    mode: '0644'
  notify: Restart Fail2ban

- name: Enable and start Fail2ban
  ansible.builtin.service:
    name: fail2ban
    enabled: true
    state: started
```

## `roles/security/templates/00-ansible-hardening.conf.j2`

```jinja
# Managed by Ansible. Do not edit manually.
Port {{ security_ssh_port }}
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 30
{% if security_ssh_allow_users | length > 0 %}
AllowUsers {{ security_ssh_allow_users | join(' ') }}
{% endif %}
```

## `roles/security/templates/sshd.local.j2`

```jinja
# Managed by Ansible. Do not edit manually.
[sshd]
enabled = true
port = {{ security_ssh_port }}
backend = systemd
bantime = {{ security_fail2ban_bantime }}
findtime = {{ security_fail2ban_findtime }}
maxretry = {{ security_fail2ban_maxretry }}
```

## `roles/users/defaults/main.yml`

```yaml
---
admin_users: []
```

## `roles/users/handlers/main.yml`

```yaml
---
[]
```

## `roles/users/meta/main.yml`

```yaml
---
galaxy_info:
  role_name: users
  author: project
  description: Administrator accounts and SSH keys
  license: MIT
  min_ansible_version: '2.16'
  platforms:
    - name: Ubuntu
      versions: [jammy, noble]
dependencies: []
```

## `roles/users/tasks/main.yml`

```yaml
---
- name: Validate administrator definitions
  ansible.builtin.assert:
    that:
      - item.name is match('^[a-z_][a-z0-9_-]*[$]?$')
      - item.ssh_keys | length > 0
      - item.ssh_keys | select('match', '^(ssh-ed25519|ssh-rsa|ecdsa-sha2-nistp256) [A-Za-z0-9+/=]+( .*)?$') | list | length == item.ssh_keys | length
    fail_msg: Each administrator needs a valid Unix name and at least one SSH public key.
  loop: "{{ admin_users }}"
  no_log: true

- name: Create administrator accounts
  ansible.builtin.user:
    name: "{{ item.name }}"
    comment: "{{ item.comment | default('System administrator') }}"
    shell: /bin/bash
    groups: sudo
    append: true
    create_home: true
    password: "{{ item.password_hash | default(omit) }}"
    update_password: always
    state: present
  loop: "{{ admin_users }}"
  no_log: true

- name: Install administrator SSH authorized keys
  ansible.posix.authorized_key:
    user: "{{ item.name }}"
    key: "{{ item.ssh_keys | join('\n') }}"
    state: present
    exclusive: true
  loop: "{{ admin_users }}"
  no_log: true

- name: Configure passwordless sudo for opted-in administrators
  ansible.builtin.copy:
    dest: "/etc/sudoers.d/90-ansible-{{ item.name }}"
    content: "{{ item.name }} ALL=(ALL:ALL) NOPASSWD:ALL\n"
    owner: root
    group: root
    mode: '0440'
    validate: /usr/sbin/visudo -cf %s
  loop: "{{ admin_users }}"
  when: item.passwordless_sudo | default(false) | bool

- name: Remove passwordless sudo for other administrators
  ansible.builtin.file:
    path: "/etc/sudoers.d/90-ansible-{{ item.name }}"
    state: absent
  loop: "{{ admin_users }}"
  when: not (item.passwordless_sudo | default(false) | bool)
```
