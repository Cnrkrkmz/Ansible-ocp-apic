# OCP 4.20.17 – VMware IPI Ansible Kurulum Scripti

SGK_v4.docx dökümanının **Bölüm 2 (OpenShift Kurulumu)** ve **Bölüm 3 (Infra Node)** adımlarını
otomatikleştiren Ansible playbook'u.

---

## Proje Yapısı

```
ocp-install-ansible/
├── site.yml                          # Ana playbook
├── inventory/
│   └── hosts.ini                     # Bastion host tanımı
├── group_vars/
│   └── all.yml                       # Tüm değişkenler
└── templates/
    ├── install-config.yaml.j2        # OCP install-config şablonu
    └── machineset-infra.yaml.j2      # Infra MachineSet şablonu
```

---

## Ön Gereksinimler

| Gereksinim | Açıklama |
|---|---|
| Bastion host | RHEL 8/9 veya Rocky Linux |
| Ansible | `>= 2.14` (`pip install ansible`) |
| Koleksiyonlar | `community.crypto`, `ansible.posix` |
| DNS | `api.<cluster>.<domain>` ve `*.apps.<cluster>.<domain>` kayıtları hazır olmalı |
| Pull Secret | https://console.redhat.com/openshift/install adresinden alınır |
| vCenter erişimi | Kurulum sırasında API erişimi açık olmalı |
| İnternet erişimi | Bastion'dan `mirror.openshift.com` erişimi gerekli |

### Gerekli Ansible koleksiyonlarını yükle

```bash
ansible-galaxy collection install community.crypto ansible.posix
```

---

## Hızlı Başlangıç

### 1. Değişkenleri düzenle

```yaml
# group_vars/all.yml
cluster_name: "ocp"
base_domain: "example.com"
pull_secret: '{"auths": ...}'   # Red Hat konsolundan al
vcenter_host: "vcenter.example.local"
vcenter_username: "administrator@vsphere.local"
vcenter_password: "CHANGE_ME"
vcenter_datacenter: "Datacenter"
vcenter_cluster: "Cluster"
vcenter_network: "VM Network"
vcenter_datastore: "datastore1"
machine_network_cidr: "10.0.0.0/16"
```

### 2. Inventory'yi düzenle

```ini
# inventory/hosts.ini
[bastion]
bastion ansible_host=<BASTION_IP> ansible_user=root
```

### 3. Playbook'u çalıştır

```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

> **Not:** Cluster kurulum adımı yaklaşık **30–60 dakika** sürer.
> Ansible `async: 7200 / poll: 60` ile her dakika durumu kontrol eder.

---

## Playbook Adımları (SGK_v4.docx Referansı)

| Adım | Ansible Görevi | Dokümandaki Karşılığı |
|------|---------------|----------------------|
| 1 | Kurulum dizini oluştur | §2.2 – `mkdir ocpinstall` |
| 2 | OCP binary'leri indir | §2.2 – `wget` ile tarball indirme |
| 3 | Arşivleri aç, PATH'e taşı | §2.2 – `tar -xvf / mv` |
| 4 | vCenter Trusted CA indir | §2.3 – CA zip indirme |
| 5 | CA sertifikaları sisteme yükle | §2.3 – `update-ca-trust extract` |
| 6 | SSH key oluştur | §2.4 – `ssh-keygen -t ed25519` |
| 7 | Hostname ayarla | §2.5 – hostname doğrulama |
| 8 | SELinux devre dışı | §2.5 – `SELINUX=disabled` |
| 9 | firewalld durdur/devre dışı | §2.5 – `systemctl disable firewalld` |
| 10 | install-config.yaml oluştur | §2.6 – config şablonu |
| 11 | Config yedeği al | §2.6 – `cp install-config.yaml ...` |
| 12 | Cluster kur | §2.6 – `openshift-install create cluster` |
| 13 | KUBECONFIG ayarla | §2.7 – `export KUBECONFIG=...` |
| 14 | Infra MachineSet oluştur | §3.1 – `oc create -f machineset-infra.yaml` |
| 15 | Infra node'larının hazır olmasını bekle | §3.1 – `oc get machines -w` |
| 16 | Default storage class değiştir | §4.4 – ceph-rbd'yi default yap |

---

## Önemli Notlar

### Infra Node Taint'leri (§9.4)
Taint'ler **kasıtlı olarak** bu playbook'a dahil edilmemiştir. Dokümanda belirtildiği gibi
ODF toleration yapılandırması tamamlandıktan sonra manuel olarak uygulanmalıdır:

```bash
export KUBECONFIG=~/ocpinstall/ocp4/auth/kubeconfig

oc adm taint nodes -l node-role.kubernetes.io/infra \
    node-role.kubernetes.io/infra=reserved:NoSchedule

oc adm taint nodes -l node-role.kubernetes.io/infra \
    node-role.kubernetes.io/infra=reserved:NoExecute

oc adm taint nodes -l node-role.kubernetes.io/infra \
    node.ocs.openshift.io/storage=true:NoSchedule
```

### VMware Ekstra Disk (§3.2)
Infra node'larına **1 TB ekstra disk** eklenmesi VMware arayüzünden yapılır
(ODF storage havuzu için). Bu adım Ansible ile otomatize edilemez.

### ODF Disk Tanıma Sorunu (§4.3)
IBM Lab ortamında eklenen diskler SSD olarak tanınmayabilir. Bu durumda
her storage node için debug pod üzerinden çalıştırın:

```bash
oc debug node/<node-name>
chroot /host
for disk in $(ls /sys/block/ | grep -E '^sd|^vd'); do
  echo 0 > /sys/block/$disk/queue/rotational
done
```

---

## Güvenlik

`group_vars/all.yml` içindeki hassas veriler (`vcenter_password`, `pull_secret`)
**Ansible Vault** ile şifrelenmelidir:

```bash
# Vault ile şifreli değişken dosyası oluştur
ansible-vault encrypt group_vars/all.yml

# Vault şifresiyle çalıştır
ansible-playbook -i inventory/hosts.ini site.yml --ask-vault-pass
```
