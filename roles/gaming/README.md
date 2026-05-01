# Role: Gaming

Esta role configura o ambiente para jogos, instalando Steam e ferramentas relacionadas via Flatpak.

## Descrição

A role `gaming` instala e configura o Steam e ferramentas de suporte para jogos no sistema, utilizando Flatpak para isolamento e facilidade de gerenciamento.

## Requisitos

- Flatpak instalado no sistema
- Acesso à internet para baixar aplicativos do Flathub
- Espaço em disco suficiente para jogos (recomendado: 100GB+)

## Variáveis

Definidas em `defaults/main.yml`:

```yaml
gaming_flatpaks:
  - com.valvesoftware.Steam      # Cliente Steam
  - com.github.tchx84.Flatseal   # Gerenciador de permissões Flatpak
```

## Estrutura

```
roles/gaming/
├── defaults/
│   └── main.yml              # Variáveis padrão
├── tasks/
│   ├── main.yml              # Ponto de entrada
│   ├── common_configs.yml    # Configurações comuns
│   ├── fedora.yml            # Tarefas específicas do Fedora
│   ├── opensuse_tumbleweed.yml
│   └── opensuse_microos.yml
└── README.md                 # Esta documentação
```

## Funcionamento

### 1. Adicionar Repositório Flathub

A role adiciona o repositório Flathub como **user-level** (não system-wide):

```bash
flatpak remote-add --user --if-not-exists flathub \
  https://dl.flathub.org/repo/flathub.flatpakrepo
```

**Vantagens do user-level:**
- ✅ Não requer privilégios de root
- ✅ Isolado por usuário
- ✅ Fácil de remover
- ✅ Não afeta outros usuários do sistema

### 2. Instalar Aplicativos via Flatpak

#### a) **Steam**
```bash
flatpak install --user flathub com.valvesoftware.Steam
```

**Recursos:**
- Cliente completo do Steam
- Suporte a Proton para jogos Windows
- Steam Play habilitado
- Overlay do Steam
- Workshop e Community

#### b) **Flatseal**
```bash
flatpak install --user flathub com.github.tchx84.Flatseal
```

**Recursos:**
- Interface gráfica para gerenciar permissões de Flatpaks
- Controle de acesso a arquivos, rede, dispositivos
- Útil para dar ao Steam acesso a discos externos
- Debugging de problemas de permissão

### 3. Otimizações para Gaming

A role fornece dicas para otimização, incluindo o uso do **GameMode**.

#### GameMode

O GameMode otimiza o sistema para jogos:
- Ajusta prioridade de CPU
- Desabilita compositing temporariamente
- Ajusta governor da CPU
- Melhora performance geral

**Uso no Steam:**
```
Propriedades do jogo → Opções de inicialização:
gamemoderun %command%
```

## Uso

### Executar a Role

```bash
# Executar apenas gaming
ansible-playbook -i inventory.yml site.yml --tags gaming
```

### Verificar Instalação

```bash
# Listar Flatpaks instalados
flatpak list --user

# Verificar Steam
flatpak info com.valvesoftware.Steam

# Verificar Flatseal
flatpak info com.github.tchx84.Flatseal
```

### Iniciar Steam

```bash
# Via linha de comando
flatpak run com.valvesoftware.Steam

# Ou pelo menu de aplicativos
# Procure por "Steam" no launcher
```

### Configurar Steam

#### Primeira Execução

1. **Login:** Entre com sua conta Steam
2. **Proton:** Habilite Steam Play para todos os títulos
   - Settings → Steam Play → Enable Steam Play for all other titles
3. **Biblioteca:** Adicione pastas de biblioteca se necessário
   - Settings → Storage → Add Drive

#### Habilitar Proton

Proton permite rodar jogos Windows no Linux:

```
Steam → Settings → Steam Play
☑ Enable Steam Play for supported titles
☑ Enable Steam Play for all other titles
Proton Version: Proton Experimental (recomendado)
```

### Gerenciar Permissões com Flatseal

```bash
# Abrir Flatseal
flatpak run com.github.tchx84.Flatseal
```

**Permissões comuns para Steam:**

1. **Acesso a Discos Externos:**
   - Filesystem → Other files: `/run/media`
   - Ou caminho específico: `/mnt/games`

2. **Acesso a Controladores:**
   - Device access → All devices (já habilitado por padrão)

3. **Rede:**
   - Network (já habilitado por padrão)

## Personalização

### Adicionar Mais Jogos/Ferramentas

Edite `roles/gaming/defaults/main.yml`:

```yaml
gaming_flatpaks:
  - com.valvesoftware.Steam
  - com.github.tchx84.Flatseal
  # Adicione mais aplicativos aqui
  - com.heroicgameslauncher.hgl    # Epic Games, GOG
  - org.prismlauncher.PrismLauncher # Minecraft
  - com.discordapp.Discord          # Discord
  - com.obsproject.Studio           # OBS Studio
```

### Configurar Biblioteca em Disco Externo

```bash
# 1. Montar disco externo
sudo mkdir -p /mnt/games
sudo mount /dev/sdX1 /mnt/games

# 2. Dar permissão ao Steam via Flatseal
# Filesystem → Other files: /mnt/games

# 3. Adicionar no Steam
# Settings → Storage → Add Drive → /mnt/games
```

### Otimizações Adicionais

#### 1. Instalar GameMode (se não estiver instalado)

```bash
# openSUSE Tumbleweed/MicroOS
sudo zypper in gamemode

# Fedora
sudo dnf install gamemode

# Verificar se está rodando
systemctl --user status gamemoded
```

#### 2. Configurar Kernel para Gaming

```bash
# Adicionar parâmetros de boot
sudo nano /etc/default/grub

# Adicionar à linha GRUB_CMDLINE_LINUX_DEFAULT:
mitigations=off processor.max_cstate=1

# Atualizar GRUB
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 3. Ajustar Swappiness

```bash
# Reduzir uso de swap para melhor performance
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Troubleshooting

### Problema: Steam não inicia

**Sintoma:**
```
Steam não abre ou fecha imediatamente
```

**Diagnóstico:**
```bash
# Executar com logs
flatpak run com.valvesoftware.Steam 2>&1 | tee steam.log

# Verificar permissões
flatpak info --show-permissions com.valvesoftware.Steam
```

**Soluções:**

1. **Limpar cache do Steam:**
   ```bash
   rm -rf ~/.var/app/com.valvesoftware.Steam/.local/share/Steam
   flatpak run com.valvesoftware.Steam
   ```

2. **Reinstalar:**
   ```bash
   flatpak uninstall --user com.valvesoftware.Steam
   flatpak install --user flathub com.valvesoftware.Steam
   ```

3. **Verificar drivers gráficos:**
   ```bash
   glxinfo | grep "OpenGL renderer"
   vulkaninfo | grep "deviceName"
   ```

### Problema: Jogos não iniciam

**Sintoma:**
```
Jogo não inicia ou trava na tela de loading
```

**Diagnóstico:**
```bash
# Ver logs do Proton
cat ~/.var/app/com.valvesoftware.Steam/.local/share/Steam/steamapps/compatdata/<APPID>/pfx/drive_c/users/steamuser/Temp/proton_log.txt
```

**Soluções:**

1. **Trocar versão do Proton:**
   - Propriedades do jogo → Compatibility
   - Testar: Proton Experimental, Proton 8.0, Proton GE

2. **Adicionar variáveis de ambiente:**
   ```
   Opções de inicialização:
   PROTON_USE_WINED3D=1 %command%
   PROTON_NO_ESYNC=1 %command%
   DXVK_ASYNC=1 %command%
   ```

3. **Verificar ProtonDB:**
   - Acesse: https://www.protondb.com/
   - Busque o jogo
   - Veja configurações recomendadas

### Problema: Performance ruim

**Sintoma:**
```
FPS baixo, stuttering, lag
```

**Soluções:**

1. **Usar GameMode:**
   ```
   Opções de inicialização: gamemoderun %command%
   ```

2. **Habilitar Mangohud (overlay de FPS):**
   ```bash
   flatpak install --user flathub org.freedesktop.Platform.VulkanLayer.MangoHud
   
   # Opções de inicialização:
   mangohud %command%
   ```

3. **Ajustar configurações gráficas:**
   - Reduzir resolução
   - Desabilitar anti-aliasing
   - Reduzir qualidade de texturas

4. **Verificar drivers:**
   ```bash
   # NVIDIA
   nvidia-smi
   
   # AMD
   glxinfo | grep "OpenGL renderer"
   ```

### Problema: Sem acesso a disco externo

**Sintoma:**
```
Steam não vê biblioteca em disco externo
```

**Solução:**
```bash
# 1. Abrir Flatseal
flatpak run com.github.tchx84.Flatseal

# 2. Selecionar Steam
# 3. Filesystem → Other files
# 4. Adicionar: /run/media ou /mnt/games

# 5. Reiniciar Steam
flatpak kill com.valvesoftware.Steam
flatpak run com.valvesoftware.Steam
```

### Problema: Controlador não funciona

**Sintoma:**
```
Controle não é detectado no jogo
```

**Soluções:**

1. **Verificar detecção:**
   ```bash
   # Listar dispositivos de entrada
   ls -la /dev/input/
   
   # Testar com jstest
   jstest /dev/input/js0
   ```

2. **Habilitar Steam Input:**
   - Steam → Settings → Controller
   - General Controller Settings
   - ☑ PlayStation/Xbox/Generic Configuration Support

3. **Dar permissão ao Flatpak:**
   ```bash
   # Via Flatseal
   Device access → All devices: ON
   ```

### Problema: Áudio não funciona

**Sintoma:**
```
Sem som nos jogos
```

**Soluções:**

1. **Verificar PulseAudio/PipeWire:**
   ```bash
   pactl list sinks
   ```

2. **Reiniciar serviço de áudio:**
   ```bash
   systemctl --user restart pipewire pipewire-pulse
   ```

3. **Verificar permissões:**
   ```bash
   # Via Flatseal
   Socket access → PulseAudio: ON
   ```

## Jogos Recomendados para Testar

### Nativos Linux (Funcionam perfeitamente)

- **Counter-Strike 2** (Free)
- **Dota 2** (Free)
- **Team Fortress 2** (Free)
- **Portal 2**
- **Left 4 Dead 2**

### Via Proton (Jogos Windows)

- **Elden Ring** (Platinum no ProtonDB)
- **Cyberpunk 2077** (Gold no ProtonDB)
- **Red Dead Redemption 2** (Gold no ProtonDB)
- **The Witcher 3** (Platinum no ProtonDB)

### Verificar Compatibilidade

Antes de comprar, verifique em:
- **ProtonDB:** https://www.protondb.com/
- **AreWeAntiCheatYet:** https://areweanticheatyet.com/

## Ferramentas Adicionais

### Lutris (Gerenciador de Jogos)

```bash
flatpak install --user flathub net.lutris.Lutris
```

**Recursos:**
- Suporte a GOG, Epic, Origin, Uplay
- Scripts de instalação automática
- Wine configurado automaticamente

### Heroic Games Launcher

```bash
flatpak install --user flathub com.heroicgameslauncher.hgl
```

**Recursos:**
- Epic Games Store
- GOG
- Amazon Prime Gaming
- Interface moderna

### Discord

```bash
flatpak install --user flathub com.discordapp.Discord
```

**Recursos:**
- Chat de voz/texto
- Overlay no jogo
- Compartilhamento de tela

## Boas Práticas

### 1. Organização de Bibliotecas

```
/mnt/games/
├── SteamLibrary/
│   ├── steamapps/
│   └── workshop/
├── GOG/
└── Epic/
```

### 2. Backup de Saves

```bash
# Saves do Steam (Flatpak)
~/.var/app/com.valvesoftware.Steam/.local/share/Steam/userdata/

# Backup
tar -czf steam-saves-$(date +%Y%m%d).tar.gz \
  ~/.var/app/com.valvesoftware.Steam/.local/share/Steam/userdata/
```

### 3. Atualização Regular

```bash
# Atualizar todos os Flatpaks
flatpak update --user

# Atualizar apenas Steam
flatpak update --user com.valvesoftware.Steam
```

### 4. Limpeza de Espaço

```bash
# Remover dados não utilizados do Flatpak
flatpak uninstall --unused --user

# Limpar cache do Steam
rm -rf ~/.var/app/com.valvesoftware.Steam/.local/share/Steam/appcache/
```

## Recursos Adicionais

### Links Úteis

- **ProtonDB:** https://www.protondb.com/
- **Steam Deck Verified:** https://www.steamdeck.com/verified
- **Flathub:** https://flathub.org/
- **GameMode:** https://github.com/FeralInteractive/gamemode
- **MangoHud:** https://github.com/flightlessmango/MangoHud

### Comunidades

- **r/linux_gaming:** Reddit
- **GamingOnLinux:** https://www.gamingonlinux.com/
- **Boiling Steam:** https://boilingsteam.com/

## Comparação: Flatpak vs. Nativo

| Aspecto | Instalação Nativa | Flatpak |
|---------|-------------------|---------|
| **Isolamento** | Acesso total ao sistema | Sandboxed |
| **Dependências** | Podem conflitar | Isoladas |
| **Atualizações** | Via gerenciador de pacotes | Via Flatpak |
| **Permissões** | Todas | Configuráveis |
| **Performance** | Nativa | ~99% nativa |
| **Portabilidade** | Dependente da distro | Funciona em todas |

## Notas Importantes

1. **Drivers Gráficos:** Certifique-se de ter os drivers mais recentes instalados.

2. **Espaço em Disco:** Jogos modernos podem ocupar 50-100GB cada.

3. **Proton:** Nem todos os jogos funcionam perfeitamente. Verifique ProtonDB.

4. **Anti-Cheat:** Alguns jogos com anti-cheat não funcionam no Linux.

5. **Performance:** Jogos nativos geralmente têm melhor performance que via Proton.

6. **Mods:** Alguns mods podem não funcionar corretamente via Proton.

## Suporte

Para problemas ou dúvidas:
1. Consulte a seção de Troubleshooting acima
2. Verifique ProtonDB para o jogo específico
3. Comunidade GamingOnLinux
4. Fórum do Steam
5. Abra uma issue no repositório do projeto

## Licença

Este código é fornecido como está, sem garantias. Use por sua conta e risco.