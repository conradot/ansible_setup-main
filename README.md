# 🚀 Kalpa Automation Station

Este projeto automatiza a configuração de uma estação de trabalho completa no **openSUSE Kalpa** (distribuição imutável). Ele utiliza uma arquitetura baseada em **Podman**, **Distrobox**, **Flatpak** e **Systemd User Services** para manter o sistema base limpo.

---

## 🏗️ Arquitetura do Projeto

A configuração é dividida em camadas para garantir isolamento e performance:

*   **Host (Nativo):** Configurações de hardware (I2C/Brilho), Bootloader, Kernel e SSD/TPM.
*   **Infrastructure (Podman):** Containers de IA (Ollama, Qdrant, Open WebUI).
*   **Workspaces (Distrobox):**
    *   `ansible-box`: Container dedicado para rodar este projeto e outras automações.
    *   `dev-workspace`: Ambiente de desenvolvimento (VS Code, Git, Compiladores).
    *   `banking-workspace`: Ambiente isolado para o Warsaw/Itaú.
*   **Desktop (Flatpak):** Aplicações GUI (Steam, Chrome, Spotify).

---

## 📂 Estrutura de Pastas

```text
.
├── roles/
│   ├── host_config/       # Hardware, Bootloader, Kernel, TPM2
│   ├── ansible_box/       # Container Distrobox para rodar o Ansible
│   ├── ia_containers/     # Stack de IA (Podman)
│   ├── dev_env/           # Workspace de Desenvolvimento (Distrobox Assemble)
│   ├── banking/           # Workspace Bancário + Warsaw (Distrobox Assemble)
│   ├── gaming/            # Steam + Flatpak
│   ├── flatpak/           # Apps (Chrome, Spotify, etc.)
│   ├── rclone/            # Backup e Sincronismo (Gdrive)
│   └── zsh/               # Configuração de Shell
├── site.yml               # Playbook mestre
└── inventory.yml          # Inventário (Conexão SSH Local)
```

---

## 🛠️ Como Iniciar (Bootstrap)

Existem duas formas de rodar este projeto pela primeira vez:

### Opção A: Direto no Host (via Venv)
Ideal para a primeira execução, quando você ainda não tem o `ansible-box`.

```bash
# 1. Criar e ativar venv
python3 -m venv .venv
source .venv/bin/activate

# 2. Instalar Ansible e coleções
pip install ansible
ansible-galaxy collection install containers.podman community.general ansible.posix

# 3. Rodar o setup do host e do ansible-box
ansible-playbook site.yml --tags host,ansible
```

### Opção B: Via Ansible-Box (Recomendado)
Após a primeira execução da Opção A, você terá um container dedicado.

```bash
# 1. Entrar no container
distrobox enter ansible-box

# 2. Rodar o playbook (o Ansible já está instalado lá)
ansible-playbook site.yml
```

---

## 🚀 Uso e Tags

Você pode rodar partes específicas do setup usando tags:

| Tag | Descrição |
| :--- | :--- |
| `host` | Configurações de Hardware, Boot, Kernel e TPM2 |
| `ansible` | Cria/Atualiza o container `ansible-box` |
| `ia` | Sobe a stack de IA (Ollama, etc) |
| `dev` | Configura o ambiente de desenvolvimento |
| `bank` | Configura o ambiente bancário e Warsaw |
| `flatpak`| Instala aplicações Desktop |
| `rclone` | Configura backups e sincronismo |

---

## 💡 Notas Importantes

### SSH Nativa
O projeto utiliza `ansible_connection: ssh` para se comunicar com o `localhost`. Para que isso funcione sem pedir senha a cada comando, configure seu acesso SSH local:

1.  **Gerar Chave (caso não tenha):**
    ```bash
    ssh-keygen -t ed25519 -C "seu-email@exemplo.com"
    ```
2.  **Autorizar Chave e Habilitar Serviço:**
    ```bash
    # Adicionar sua chave à lista de autorizados
    cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys

    # Ativar o serviço SSH no Host
    sudo systemctl enable --now sshd
    ```

### Distrobox Assemble
As roles de workspace (`dev_env`, `ansible_box`, `banking`) utilizam o `distrobox assemble`. Isso significa que as configurações são declarativas e gerenciadas via arquivos `.ini` em `~/.config/distrobox/`.

### Host Config & Reboot
Alterações em `host_config` (como parâmetros do kernel e pacotes de hardware) só entram em vigor após um **reboot**, pois o Kalpa utiliza o modelo de atualização transacional.

---
**Dica:** Sempre mantenha este repositório sincronizado. Ele é o "cérebro" da sua máquina! 🧠⚡
