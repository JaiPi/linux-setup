# linux-setup

Gets me up and running on any new Linux machine: **Debian, Fedora and WSL**. This Ansible playbook installs essential packages, applies my dotfiles, and sets up Zsh and Terraform tooling.

## 🚀 Features
- **System Package Installation** (`apt` for Debian, `dnf` for Fedora)
- **Dotfiles Management** (using `stow`)
- **WSL-Specific Configurations** (if applicable)
- **Terraform Tools Setup**
  - [terraform-docs](https://terraform-docs.io/)
  - [tfswitch](https://tfswitch.warrensbox.com/)
- **Ensures Required Directories Exist** (`~/.local/bin`, `~/repos`)

---

## 🛠️ **Prerequisites**
Before running the playbook, ensure you have:
- **Ansible installed** 
  - Debian: `sudo apt update && sudo apt install -y ansible python3-apt` 
  - Fedora: `sudo dnf install -y ansible python3-dnf`

---

## 📥 **Installation**
Clone this repository and navigate to the playbook directory:
```sh
git clone git@github.com:JaiPi/linux-setup.git
cd linux-setup
```

---

## ▶️ Usage
Run the playbook with:

```sh
ansible-playbook playbook.yml --ask-become-pass
```

# 📌 Available Run Modes

| **Mode**           | **Command**                                             |
|--------------------|---------------------------------------------------------|
| 🏃 **Run normally** | `ansible-playbook playbook.yml --ask-become-pass`       |
| 🔍 **Dry-run** *(Check mode)* | `ansible-playbook playbook.yml --ask-become-pass --check` |
| 🔧 **Verbose mode** | `ansible-playbook playbook.yml --ask-become-pass -vvvv` |
| 🎯 **Run a single role** | `ansible-playbook playbook.yml --ask-become-pass --tags stow` |
