# OCP 4.20.17 + ODF – VMware IPI Ansible Kurulum Scripti

SGK_v5.docx dökümanının **Bölüm 2 (OCP Kurulumu)**, **Bölüm 3 (Infra Node)**
ve **Bölüm 4 (ODF)** adımlarını otomatikleştiren Ansible playbook seti.

---

## Proje Yapısı

```
Ansible-ocp-apic/
├── site.yml                    # Ana giriş noktası – tüm süreci orkestre eder
├── plays/
│   └── 04_odf.yml              # §4 – ODF + Local Storage Operator kurulumu
├── inventory/
│   └── hosts.ini               # Bastion host tanımı
├── group_vars/
│   └── all.yml                 # Tüm değişkenler (buraya secret yazmayın)
├── templates/
│   ├── install-config.yaml.j2
│   └── machineset-infra.yaml.j2
└── requirements.yml            # Ansible koleksiyon gereksinimleri
```

---

## Ön Gereksinimler

### 1. Altyapı gereksinimleri

| Gereksinim | Detay |
|---|---|
| Bastion host | RHEL 8 üzerinde çalışan bir VM |
| Ansible | `>= 2.14` — bastion üzerinde kurulu olmalı |
| Python | `>= 3.8` — `pip` ile paket kurulabilmeli |
| DNS kayıtları | `api.<cluster>.<domain>` ve `*.apps.<cluster>.<domain>` açık olmalı |
| Pull Secret | https://console.redhat.com/openshift/install adresinden alınır |
| vCenter erişimi | Bastion → vCenter API (443) açık olmalı |
| İnternet erişimi | Bastion → `mirror.openshift.com` açık olmalı |
| vmware-ipi.yaml | `/home/admin/vmware-ipi.yaml` dosyası bastion üzerinde mevcut olmalı |

### 2. vmware-ipi.yaml formatı

Playbook bu dosyayı otomatik okur; vCenter/cluster bilgilerini buraya yazın:

```yaml
# /home/admin/vmware-ipi.yaml
vsphere_hostname:      "vcenter.example.local"
vsphere_username:      "administrator@vsphere.local"
vsphere_password:      "CHANGE_ME"
vsphere_datacenter:    "Datacenter"
vsphere_cluster:       "Cluster"
vsphere_network:       "VM Network"
vsphere_datastore:     "datastore1"
vsphere_resource_pool: "/Datacenter/host/Cluster/Resources/ocp"
vsphere_folder:        "/Datacenter/vm/ocp"
cluster_name:          "ocp"
base_domain:           "example.com"
vsphere_api_vip:       "192.168.252.5"
vsphere_ingress_vip:   "192.168.252.6"
```

---

## Kurulum – Adım Adım

### Adım 1 — Koleksiyonları ve bağımlılıkları yükle

Bastion host üzerinde bir kez çalıştırın:

```bash
# Ansible koleksiyonları
ansible-galaxy collection install -r requirements.yml

# community.vmware için Python kütüphanesi
pip install pyVmomi
```

### Adım 2 — Inventory'yi kontrol et

`inventory/hosts.ini` iki senaryo içerir:

```ini
# Senaryo A: Ansible bastion üzerinde çalışıyorsa (önerilen)
[bastion]
bastion ansible_host=127.0.0.1 ansible_connection=local ansible_user=root

# Senaryo B: Ansible kendi bilgisayarınızda kuruluysa
# [bastion]
# bastion ansible_host=<BASTION_IP> ansible_user=root
```

Durumunuza göre ilgili satırı aktif bırakın, diğerini yorum yapın.

### Adım 3 — group_vars/all.yml'i gözden geçir

```yaml
# Versiyon ve dizinler otomatik gelir, değiştirmenize gerek yok.
# Sadece IBM ortamına özel değerleri kontrol edin:
machine_network_cidr: "192.168.252.0/24"   # sabit
infra_extra_disk_gb:  1024                  # ODF için her infra node'una eklenecek disk
odf_channel:          "stable-4.16"        # OCP 4.20 uyumlu
```

### Adım 4 — Playbook'u çalıştır

```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

Playbook başlarken Pull Secret'ı terminalden sorar (ekranda görünmez):

```
Red Hat Pull Secret
(https://console.redhat.com/openshift/install adresinden alın):
```

> **Toplam süre:** ~2–2.5 saat  
> (OCP cluster kurulumu ~60 dk, ODF StorageCluster Ready ~30 dk)

---

## Playbook Akışı

`site.yml` tek komutla aşağıdaki tüm adımları sırayla işler:

```
site.yml
│
├── §2.1  Root geçişi          (become: true ile otomatik)
├── §2.2  OCP binary indirme   wget → tar → /usr/local/bin
├── §2.3  vCenter CA sertifika unzip → update-ca-trust
├── §2.4  SSH key oluştur      ssh-keygen ed25519
├── §2.5  Sistem hazırlığı     SELinux=disabled, firewalld durdur
├── §2.6  install-config.yaml  OCP 4.14+ failureDomains formatı
├── §2.6  Cluster kur          openshift-install create cluster (~60 dk)
├── §2.7  KUBECONFIG ayarla    .bashrc + /etc/profile.d
│
├── §3.1  Infra MachineSet     worker YAML → python3 patch → oc create
├── §3.1  Infra node bekle     machines Running olana kadar (maks 30 dk)
│
├── §3.2  Disk ekle (OTOMATİK) community.vmware ile:
│         VM kapat → 1 TB disk ekle → VM başlat → node Ready bekle
│
└── import_playbook: plays/04_odf.yml
    ├── §4.1  ODF Operatör     Namespace + OperatorGroup + Subscription
    │         InstallPlan approve → CSV Succeeded bekle
    ├── §4.1  Local Storage Op aynı akışla
    ├── §4.3  Rotational fix   oc debug node → /sys/block/*/rotational=0
    ├── §4.2  LocalVolumeSet   local-block → PV oluşmasını bekle
    ├── §4.2  StorageCluster   ocs-storagecluster → Ready bekle (~30 dk)
    └── §4.4  Default SC       thin-csi → ocs-storagecluster-ceph-rbd
```

**Bitince:** "ODF kurulumu başarıyla tamamlandı. Sonraki adım: §5 CP4I Catalog" mesajını görürsünüz.

---

## Sık Kullanılan Komutlar

```bash
# Sadece belirli bir adımdan itibaren çalıştır (tag eklenirse)
ansible-playbook -i inventory/hosts.ini site.yml --start-at-task "ODF Operatör..."

# Dry-run (değişiklik yapmadan ne yapacağını göster)
ansible-playbook -i inventory/hosts.ini site.yml --check

# Verbose çıktı (sorun gidermek için)
ansible-playbook -i inventory/hosts.ini site.yml -vv

# Sadece ODF adımlarını çalıştır
ansible-playbook -i inventory/hosts.ini plays/04_odf.yml
```

---

## Güvenlik

`group_vars/all.yml` dosyasına **asla** şifre veya secret yazmayın.  
Pull secret playbook başında terminalden sorulur.  
vCenter şifresi `/home/admin/vmware-ipi.yaml` üzerinde tutulur — bu dosyayı `600` izniyle koruyun:

```bash
chmod 600 /home/admin/vmware-ipi.yaml
```

Vault kullanmak isterseniz:

```bash
ansible-vault encrypt /home/admin/vmware-ipi.yaml
ansible-playbook -i inventory/hosts.ini site.yml --ask-vault-pass
```

---

## Adım Tablosu (SGK_v5 Referansı)

| Adım # | SGK_v5 § | Playbook Görevi |
|--------|----------|----------------|
| 1–4    | §2.2     | OCP binary indirme ve PATH |
| 5–6    | §2.3     | vCenter CA sertifikası |
| 7      | §2.4     | SSH key oluşturma |
| 8–10   | §2.5     | Hostname / SELinux / Firewall |
| 11–13  | §2.6     | install-config + cluster kurulumu |
| 14     | §2.7     | KUBECONFIG |
| 15–17  | §3.1     | Infra MachineSet |
| 18     | §3.2     | Otomatik disk ekleme (vmware_guest_disk) |
| 19     | §4.1     | ODF Operatör kurulumu |
| 20     | §4.1     | Local Storage Operator |
| 21     | §4.3     | Disk rotational fix |
| 22     | §4.2     | LocalVolumeSet + StorageCluster |
| 23     | §4.4     | Default StorageClass değişimi |
