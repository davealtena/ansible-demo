# Ansible Baseline + Audit Demo

Semaphore demo project: een Ansible baseline role die Debian servers configureert en een audit playbook dat compliance controleert.

## Wat doet dit project?

- **Baseline role** - Configureert een Debian server met:
  - NTP (chrony) met Nederlandse pool servers
  - SSH hardening (geen wachtwoord login, max 3 pogingen)
  - Firewall (UFW) met default deny incoming
  - Custom MOTD

- **Audit playbook** - Controleert of alle baseline instellingen correct zijn en rapporteert resultaten

- **Molecule tests** - Geautomatiseerde tests in Docker containers
- **GitHub Actions CI** - Automatische lint checks en Molecule tests bij elke push/PR

## Vereisten

- Python 3.10+
- Ansible 2.15+
- Docker (voor Molecule tests)
- SSH toegang tot de target server

### Installatie

```bash
# Maak een virtual environment
python3 -m venv venv
source venv/bin/activate

# Installeer dependencies
pip install ansible molecule molecule-plugins[docker] ansible-lint yamllint docker

# Installeer pre-commit hooks
pip install pre-commit
pre-commit install

# Installeer Ansible collections
ansible-galaxy install -r requirements.yml
```

## Gebruik

### Baseline toepassen

```bash
# Dry-run (check mode)
ansible-playbook -i inventory/hosts.yml playbooks/baseline.yml --check --diff

# Uitvoeren
ansible-playbook -i inventory/hosts.yml playbooks/baseline.yml
```

### Audit uitvoeren

```bash
ansible-playbook -i inventory/hosts.yml playbooks/audit.yml
```

### Molecule tests draaien

```bash
cd roles/baseline
molecule test        # Volledige test cyclus
molecule converge    # Alleen configuratie toepassen
molecule verify      # Alleen verificatie draaien
molecule destroy     # Container opruimen
```

### Linting

```bash
yamllint .
ansible-lint
```

## Projectstructuur

```
├── .github/workflows/ci.yml    # GitHub Actions CI pipeline
├── .pre-commit-config.yaml      # Pre-commit hooks configuratie
├── .ansible-lint                # Ansible-lint configuratie
├── .yamllint                    # YAML lint configuratie
├── ansible.cfg                  # Ansible configuratie
├── requirements.yml             # Galaxy collections
├── inventory/
│   └── hosts.yml                # Inventory (192.168.1.95)
├── roles/
│   └── baseline/
│       ├── defaults/main.yml    # Configureerbare variabelen
│       ├── handlers/main.yml    # Service handlers
│       ├── tasks/               # Taken per categorie
│       ├── templates/           # Jinja2 templates
│       ├── meta/main.yml        # Role metadata
│       └── molecule/            # Molecule test configuratie
└── playbooks/
    ├── baseline.yml             # Past baseline toe
    └── audit.yml                # Controleert compliance
```

## Proxmox LXC Container Setup

Maak een Debian 12 container aan op Proxmox:

```bash
# Op de Proxmox host
pct create 100 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname semaphore-demo \
  --memory 512 --swap 256 --cores 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.95/24,gw=192.168.1.1 \
  --storage local-lvm \
  --rootfs local-lvm:4 \
  --start 1

# SSH key deployen
pct exec 100 -- mkdir -p /root/.ssh
pct push 100 ~/.ssh/id_ed25519.pub /root/.ssh/authorized_keys
pct exec 100 -- chmod 600 /root/.ssh/authorized_keys

# Python installeren (nodig voor Ansible)
pct exec 100 -- apt-get update
pct exec 100 -- apt-get install -y python3

# Snapshot maken (voor reset na demo)
pct snapshot 100 pre-baseline --description "Schone staat voor demo"
```

### Container resetten na demo

```bash
pct rollback 100 pre-baseline
pct start 100
```

## Semaphore Configuratie

1. **Project aanmaken** - Koppel aan deze Git repository
2. **Key Store** - Voeg SSH private key toe (voor toegang tot 192.168.1.95)
3. **Inventory** - Maak inventory aan, type "File", pad: `inventory/hosts.yml`
4. **Task Templates**:

| Template | Playbook | Schedule | Beschrijving |
|----------|----------|----------|-------------|
| Baseline | `playbooks/baseline.yml` | - | Handmatig uitvoeren |
| Audit | `playbooks/audit.yml` | `*/15 * * * *` | Elke 15 minuten |

## Configuratie aanpassen

Alle instellingen staan in `roles/baseline/defaults/main.yml`:

| Variabele | Default | Beschrijving |
|-----------|---------|-------------|
| `baseline_ssh_permit_root_login` | `prohibit-password` | Root login beleid |
| `baseline_ssh_password_authentication` | `no` | Wachtwoord authenticatie |
| `baseline_ssh_max_auth_tries` | `3` | Max login pogingen |
| `baseline_firewall_enabled` | `true` | UFW aan/uit |
| `baseline_motd_message` | `Beheerd door Ansible via Semaphore` | MOTD tekst |

## CI/CD

Bij elke push en pull request draait GitHub Actions:
1. **Lint** - yamllint + ansible-lint
2. **Molecule** - Volledige test cyclus in Docker
