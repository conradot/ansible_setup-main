# 🚀 Kalpa Automation Station

Este projeto automatiza a configuração de uma estação de trabalho completa no **openSUSE Kalpa / Tumbleweed**. Ele utiliza uma arquitetura baseada em **Podman**, **Distrobox**, **Flatpak** e **Systemd User Services** para manter o sistema base limpo.

---

## 🏗️ Arquitetura do Projeto

A configuração é dividida em camadas para garantir isolamento e performance:

*   **Host (Nativo):** Instalação de pacotes base (zsh, git) e configuração de hardware (I2C/Brilho do monitor).
*   **Infrastructure (Podman):** Containers de IA (Ollama, Qdrant, Open WebUI).
*   **Workspaces (Distrobox):**
    *   `dev_fedora`: Ambiente de desenvolvimento (VS Code, Git, Compiladores) e container principal utilizado para rodar este próprio playbook.
    *   `banking-workspace`: Ambiente isolado para o Warsaw/Itaú.
*   **Desktop (Flatpak):** Aplicações GUI (Steam, Chrome, Spotify, KDE Apps).

---

## 📂 Estrutura de Pastas

```text
.
├── first-run.ssh          # Script de Bootstrap para criar o dev_fedora e rodar o playbook
├── roles/
│   ├── host_config/       # Pacotes Base (Zsh, Git) e Hardware (I2C)
│   ├── ia_containers/     # Stack de IA (Podman)
│   ├── dev_env/           # Workspace de Desenvolvimento (Distrobox Assemble)
│   ├── banking/           # Workspace Bancário + Warsaw (Distrobox Assemble)
│   ├── gaming/            # Steam + Flatpak
│   ├── flatpak/           # Apps (Chrome, Spotify, etc.)
│   ├── rclone/            # Backup e Sincronismo (Gdrive)
│   └── zsh/               # Configuração do ambiente de usuário Zsh
├── site.yml               # Playbook mestre
└── inventory.yml          # Inventário (Conexão SSH Local)
```

---

## 🛠️ Como Iniciar (Bootstrap)

Para facilitar a primeira execução do projeto, disponibilizamos um script de automação (`first-run.ssh`). Ele cuida de criar seu contêiner de desenvolvimento (`dev_fedora`) e executar o Ansible por lá.

```bash
# 1. Certifique-se que o serviço SSH do Host está ativo:
sudo systemctl enable --now sshd

# 2. Torne o script executável
chmod +x first-run.ssh

# 3. Execute o bootstrap
./first-run.ssh
```

**O que o `first-run.ssh` faz?**
1. Cria e entra no contêiner `dev_fedora` usando Distrobox.
2. Cria um ambiente virtual Python (`.venv`) e instala o Ansible.
3. Autoriza sua chave SSH local para conectar ao host via `127.0.0.1` sem senha (necessário para que o Ansible configure o openSUSE de fora do contêiner).
4. Baixa e instala as dependências/coleções do Ansible (`requirements.yml`).
5. Executa o playbook mestre (`site.yml`).

---

## 🚀 Uso e Tags

Você pode rodar partes específicas do setup usando tags:
distrobox enter dev-fedora -- bash -c 'source .venv/bin/activate && ansible-playbook site.yml --tags ollama'


| Tag | Descrição |
| :--- | :--- |
| `host` | Pacotes base do sistema e I2C (Monitor) |
| `ia` | Sobe a stack de IA (Ollama, etc) |
| `dev` | Configura o ambiente de desenvolvimento |
| `bank` | Configura o ambiente bancário e Warsaw |
| `flatpak`| Instala aplicações Desktop |
| `rclone` | Configura backups e sincronismo |
| `zsh` | Instala plugins e o .zshrc do usuário |

---

## 💡 Notas Importantes

### SSH Nativa
O projeto utiliza `ansible_connection: ssh` para se comunicar com o host via `127.0.0.1`. O script `first-run.ssh` já configura o acesso sem senha de forma automatizada. 

### Distrobox Assemble
As roles de workspace (`dev_env`, `banking`) utilizam o `distrobox assemble`. Isso significa que as configurações são declarativas e gerenciadas via arquivos `.ini` em `~/.config/distrobox/`.

### Host Config & Reboot (No Kalpa / MicroOS)
Se você estiver rodando o sistema imutável openSUSE Kalpa (ou MicroOS), lembre-se que alterações feitas pela role `host_config` (como a instalação do Git ou Zsh via `transactional-update`) só entram em vigor após um **reboot**. Se estiver no openSUSE Tumbleweed, as mudanças são imediatas.

---
**Dica:** Sempre mantenha este repositório sincronizado. Ele é o "cérebro" da sua máquina! 🧠⚡
