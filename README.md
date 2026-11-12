# Ansible Server Provisioner

Ubuntu 22.04 ve 24.04 için rol tabanlı sunucu hazırlama projesi. Uygulama sırası `common`, `users`, `security`, `docker` şeklindedir. Gerçek bir yönetici SSH anahtarı tanımlanmadığında playbook başlamaz.

## Dosya ağacı

```text
.
├── .gitignore
├── PROJECT_FILES.md
├── ansible.cfg
├── group_vars/all.yml
├── inventory/
│   ├── production/
│   │   ├── hosts.ini
│   │   └── group_vars/{all.yml,vault.example.yml}
│   └── staging/
│       ├── hosts.ini
│       └── group_vars/{all.yml,vault.example.yml}
├── requirements.yml
├── roles/
│   ├── common/{defaults,handlers,meta,tasks}/main.yml
│   ├── users/{defaults,handlers,meta,tasks}/main.yml
│   ├── security/
│   │   ├── {defaults,handlers,meta,tasks}/main.yml
│   │   └── templates/{00-ansible-hardening.conf.j2,sshd.local.j2}
│   └── docker/
│       ├── {defaults,handlers,meta,tasks}/main.yml
│       └── templates/daemon.json.j2
└── site.yml
```

## Gereksinimler

- Kontrol makinesinde Python 3.10+ ve `ansible-core >= 2.16`.
- Hedefte Ubuntu 22.04 veya 24.04, APT erişimi, sudo yetkili başlangıç SSH hesabı ve Python 3.

```bash
python3 -m venv .venv
. .venv/bin/activate
python3 -m pip install 'ansible-core>=2.16,<2.19' ansible-lint
ansible-galaxy collection install -r requirements.yml -p collections
```

## Yapılandırma

`inventory/staging/hosts.ini` veya `inventory/production/hosts.ini` içindeki dokümantasyon için ayrılmış `192.0.2.x` adreslerini değiştirin. İlgili ortamın `group_vars/all.yml` dosyasına en az bir yönetici ve gerçek public key ekleyin:

```yaml
admin_users:
  - name: deploy
    comment: Deployment administrator
    ssh_keys:
      - 'ssh-ed25519 AAAA... operator@example'
    passwordless_sudo: true
```

Bu gösterim anahtar biçimini açıklar; `AAAA...` gerçek anahtar değildir. Her yönetici için ya `passwordless_sudo: true` ya da sudo için kullanılabilir `password_hash` zorunludur. Hash seçeneğinde parolanın operatör tarafından bilinmesi gerekir; aksi halde sudo kullanılamaz. `inventory/staging/group_vars/vault.example.yml` dosyasını `vault.yml` adıyla kopyalayın, gerçek hash'i ekleyin ve şifreleyin:

```bash
ansible-vault encrypt inventory/staging/group_vars/vault.yml
```

Ardından kullanıcı tanımında `password_hash: "{{ vault_admin_password_hash }}"` kullanın ve çalıştırırken `--ask-vault-pass` veya kurumsal vault kimlik kaynağını kullanın. `vault.yml` dosyaları `.gitignore` içindedir. Örnek dosyalar otomatik yüklenmez.

`security_allowed_tcp_ports` ve `security_allowed_udp_ports` yalnızca gerçekten gereken servis portlarını içermelidir. Docker grubuna kullanıcı otomatik eklenmez; Docker grubu root düzeyinde yetki verir.

## Çalıştırma

```bash
ansible-playbook -i inventory/staging/hosts.ini site.yml --syntax-check
ansible-playbook -i inventory/staging/hosts.ini site.yml --check --diff
ansible-playbook -i inventory/staging/hosts.ini site.yml
ansible-playbook -i inventory/production/hosts.ini site.yml --check --diff
ansible-playbook -i inventory/production/hosts.ini site.yml
```

`ansible.cfg` varsayılan olarak staging envanterini seçer; production için `-i` verin. `--check` ilk kurulumdaki tüm bağımlılıkları simüle edemeyebilir. İkinci gerçek çalıştırmanın, upstream paket güncellemesi yoksa, `changed=0` olması beklenir. `common_upgrade_packages: true` yeni paket güncellemeleri geldiğinde doğal olarak değişiklik yapar.

## SSH portunu değiştirme

Varsayılan port 22'dir. Port değişimi iki geçişte yapılır:

1. Ortam `group_vars/all.yml` dosyasında `security_ssh_port: 2222` ve `security_ssh_previous_port: 22` belirtin. Envanterdeki `ansible_port` eski portta kalsın. Playbook yeni portu ve mevcut bağlantı portunu UFW'de açık tutar.
2. Yeni portta public key ile oturum açıldığını ayrı terminalde doğrulayın. Envanterde `ansible_port=2222` yapıp playbook'u tekrar çalıştırın. İkinci geçiş eski UFW kuralını siler. Bulut güvenlik grubunda yeni portu ilk geçişten önce açın.

Ubuntu 24.04'te `ssh.socket` devre dışı bırakılır ve `ssh.service` etkinleştirilir. SSH yeniden başlatılmadan önce `sshd -t` çalışır. OpenSSH ayarları `00-ansible-hardening.conf` dosyasında tutulur; Ubuntu'nun daha sonra okunan cloud-init ayarlarından önce yüklenir. Yine de makineye özel `Include` ve `Match` kuralları varsa dağıtımdan sonra `sshd -T` ile etkili ayarları doğrulayın.

## Docker ve firewall sınırı

Docker'ın yayımladığı container portları UFW'nin gelen trafik kurallarını aşabilir. İnternete açık olmaması gereken servisleri `127.0.0.1:HOST_PORT:CONTAINER_PORT` biçiminde yayımlayın. Dış erişim gereken servisler için bulut güvenlik grubu ve `DOCKER-USER` zincirinde kural uygulayın. Docker'ın iptables yönetimini kapatmayın; container ağını bozabilir. `json-file` log sürücüsünde boyut ve dosya sınırı vardır. Yeni daemon log ayarları mevcut container'lara ancak yeniden oluşturulduklarında uygulanır.

## İşletim notları

- Anahtar, parola hash'i, vault parolası veya özel anahtarları repoya eklemeyin.
- Yönetici SSH anahtarları `exclusive: true` ile yönetilir; listede olmayan anahtarlar kaldırılır. İlk çalıştırmadan önce gerekli bütün anahtarları listeye ekleyin.
- `security_ssh_allow_users` varsayılan olarak boştur. Kısıtlamak için yalnızca erişimi doğrulanmış kullanıcıları girin.
- Yeniden başlatma gereksinimini bakım penceresinde ayrıca yönetin.
- `ansible-lint site.yml` ile kalite kontrolü yapın. Hedefte `chronyc tracking`, `ufw status`, `fail2ban-client status sshd`, `sshd -T`, `docker compose version` komutlarıyla durumu doğrulayın.

