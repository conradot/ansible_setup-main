# Role: ZSH

Esta role configura o ZSH (Z Shell) como shell padrão com plugins modernos e configuração otimizada.

## Descrição

A role `zsh` instala e configura o ZSH com plugins essenciais (autosuggestions, syntax highlighting, history search) e um arquivo `.zshrc` personalizado com keybindings, prompt Git-aware e aliases úteis.

## Requisitos

- ZSH instalado no sistema (instalado pela role `host_config`)
- Git (para baixar plugins)
- Acesso root para mudar shell padrão

## Variáveis

Esta role **não requer variáveis**. Todas as configurações são aplicadas automaticamente.

## Estrutura

```
roles/zsh/
├── tasks/
│   ├── main.yml              # Ponto de entrada
│   ├── common_configs.yml    # Configurações comuns
│   ├── fedora.yml            # Tarefas específicas do Fedora
│   ├── opensuse_tumbleweed.yml
│   └── opensuse_microos.yml
├── templates/
│   └── zshrc.j2              # Template do .zshrc
└── README.md                 # Esta documentação
```

## Funcionamento

### 1. Instalação de Plugins

A role baixa três plugins essenciais via Git:

```bash
~/.zsh-plugins/
├── zsh-autosuggestions/       # Sugestões baseadas no histórico
├── zsh-syntax-highlighting/   # Destaque de sintaxe em tempo real
└── zsh-history-substring-search/  # Busca no histórico com Ctrl+↑/↓
```

#### a) **zsh-autosuggestions**
```bash
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ~/.zsh-plugins/zsh-autosuggestions
```

**Recursos:**
- Sugere comandos enquanto você digita
- Baseado no histórico de comandos
- Aceitar sugestão: `→` (seta direita) ou `End`
- Aceitar palavra: `Ctrl+→`

#### b) **zsh-syntax-highlighting**
```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting \
  ~/.zsh-plugins/zsh-syntax-highlighting
```

**Recursos:**
- Destaca comandos válidos em verde
- Comandos inválidos em vermelho
- Caminhos existentes sublinhados
- Strings e variáveis coloridas

#### c) **zsh-history-substring-search**
```bash
git clone https://github.com/zsh-users/zsh-history-substring-search \
  ~/.zsh-plugins/zsh-history-substring-search
```

**Recursos:**
- Busca no histórico com `Ctrl+↑` e `Ctrl+↓`
- Busca por substring (não apenas início)
- Navegação intuitiva

### 2. Configuração do .zshrc

O arquivo `~/.zshrc` é gerado a partir do template `zshrc.j2` com:

#### a) **Variáveis de Ambiente**
```bash
export OLLAMA_HOST=127.0.0.1:11434
export PATH="$HOME/.local/bin:$HOME/.opencode/bin:$HOME/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"
```

#### b) **Keybindings (Modo Emacs)**
```bash
# Navegação
Home/End       # Início/fim da linha
Ctrl+←/→       # Palavra anterior/próxima
Ctrl+Backspace # Deletar palavra anterior
Ctrl+Delete    # Deletar palavra seguinte

# Histórico
Ctrl+↑/↓       # Busca substring no histórico
Page Up/Down   # Busca por início do comando

# Edição
Delete         # Deletar caractere
Insert         # Modo overwrite
```

#### c) **Opções do ZSH**
```bash
setopt auto_cd              # cd sem digitar 'cd'
setopt HIST_IGNORE_ALL_DUPS # Remove duplicatas do histórico
setopt interactivecomments  # Permite comentários no shell
setopt hist_verify          # Verifica antes de executar histórico
```

#### d) **Histórico**
```bash
HISTFILE=~/.histfile
HISTSIZE=10000      # 10.000 comandos na memória
SAVEHIST=10000      # 10.000 comandos salvos
```

#### e) **Completion (Autocompletar)**
```bash
# Case-insensitive
zstyle ':completion:*' matcher-list 'm:{a-z}={A-Z}'

# Sem menu de ciclo
zstyle ':completion:*' menu no
```

#### f) **Prompt Git-Aware**
```bash
[usuario@hostname:~/pasta] (branch*) $
```

**Elementos:**
- `usuario@hostname` - Usuário e máquina
- `~/pasta` - Diretório atual (últimos 3 níveis)
- `(branch*)` - Branch Git (asterisco se houver mudanças)
- `$` - Prompt normal (`#` se root)
- `[código]` - Código de erro (se comando falhou)

#### g) **Aliases**
```bash
# Cores
alias ls='ls --color=auto'
alias grep='grep --color=auto'

# Listagem
alias la='ls -A'        # Listar todos (exceto . e ..)
alias ll='ls -l'        # Listagem longa
alias lla='ls -lA'      # Listagem longa de todos

# Preservar metadados
alias cp='cp --preserve=all'
alias rsync='rsync --perms --xattrs --acls --times --atimes'
```

### 3. Definir ZSH como Shell Padrão

```bash
sudo chsh -s /usr/bin/zsh $USER
```

**Efeito:**
- ZSH será o shell padrão em novos logins
- Bash ainda disponível (`bash` para entrar)
- Requer logout/login para aplicar

## Uso

### Executar a Role

```bash
# Executar apenas zsh
ansible-playbook -i inventory.yml site.yml --tags zsh
```

### Aplicar Mudanças

**IMPORTANTE:** Após executar a role:

1. **Fazer logout/login** (para aplicar novo shell padrão)
   ```bash
   # Ou reiniciar
   sudo reboot
   ```

2. **Ou iniciar ZSH manualmente:**
   ```bash
   zsh
   ```

### Verificar Instalação

```bash
# Verificar shell padrão
echo $SHELL
# Deve mostrar: /usr/bin/zsh

# Verificar plugins
ls ~/.zsh-plugins/

# Verificar .zshrc
cat ~/.zshrc

# Testar autosuggestions
# Digite: git (aguarde sugestão aparecer)

# Testar syntax highlighting
# Digite comando válido (verde) ou inválido (vermelho)

# Testar history search
# Digite: git
# Pressione Ctrl+↑ (deve buscar comandos git no histórico)
```

## Personalização

### Adicionar Mais Plugins

Edite `roles/zsh/tasks/common_configs.yml`:

```yaml
- name: Baixar plugins do Zsh via Git
  git:
    repo: "{{ item.repo }}"
    dest: "{{ lookup('env', 'HOME') }}/.zsh-plugins/{{ item.name }}"
    depth: 1
  loop:
    - { name: 'zsh-autosuggestions', repo: 'https://github.com/zsh-users/zsh-autosuggestions' }
    - { name: 'zsh-syntax-highlighting', repo: 'https://github.com/zsh-users/zsh-syntax-highlighting' }
    - { name: 'zsh-history-substring-search', repo: 'https://github.com/zsh-users/zsh-history-substring-search' }
    # Adicione mais plugins aqui
    - { name: 'zsh-completions', repo: 'https://github.com/zsh-users/zsh-completions' }
```

E adicione ao `.zshrc`:
```bash
source ~/.zsh-plugins/zsh-completions/zsh-completions.plugin.zsh
```

### Modificar Prompt

Edite `roles/zsh/templates/zshrc.j2`:

```bash
# Prompt simples
PROMPT='%n@%m:%~$ '

# Prompt com cores
PROMPT='%F{green}%n%f@%F{blue}%m%f:%F{yellow}%~%f$ '

# Prompt com Git (já incluído)
local path_segment='[%f%F{yellow}%n@%m%f:%F{cyan}%(4~|…/%3~|%~)%f]%f'
local git_segment='$(parse_git_info)'
local prompt_symbol='%(?. . %F{red}[%?]%f )%(#.#.$) '
PROMPT="${path_segment}${git_segment}${prompt_symbol}"
```

### Adicionar Aliases Personalizados

Edite `roles/zsh/templates/zshrc.j2`:

```bash
# Seus aliases personalizados
alias update='sudo zypper dup'
alias clean='sudo zypper clean'
alias docker='podman'
alias k='kubectl'
alias tf='terraform'
alias vim='nvim'
```

### Mudar Keybindings

Edite `roles/zsh/templates/zshrc.j2`:

```bash
# Modo vi em vez de emacs
bindkey -v

# Ou adicionar keybindings personalizados
bindkey '^R' history-incremental-search-backward  # Ctrl+R
bindkey '^S' history-incremental-search-forward   # Ctrl+S
```

### Adicionar Funções Personalizadas

Edite `roles/zsh/templates/zshrc.j2`:

```bash
# Criar diretório e entrar nele
mkcd() {
  mkdir -p "$1" && cd "$1"
}

# Extrair qualquer arquivo compactado
extract() {
  if [ -f "$1" ]; then
    case "$1" in
      *.tar.bz2) tar xjf "$1" ;;
      *.tar.gz)  tar xzf "$1" ;;
      *.bz2)     bunzip2 "$1" ;;
      *.rar)     unrar x "$1" ;;
      *.gz)      gunzip "$1" ;;
      *.tar)     tar xf "$1" ;;
      *.tbz2)    tar xjf "$1" ;;
      *.tgz)     tar xzf "$1" ;;
      *.zip)     unzip "$1" ;;
      *.Z)       uncompress "$1" ;;
      *.7z)      7z x "$1" ;;
      *)         echo "'$1' não pode ser extraído" ;;
    esac
  else
    echo "'$1' não é um arquivo válido"
  fi
}
```

## Troubleshooting

### Problema: Shell não muda para ZSH

**Sintoma:**
```bash
$ echo $SHELL
/bin/bash
```

**Solução:**
```bash
# Verificar se ZSH está instalado
which zsh

# Mudar shell manualmente
sudo chsh -s /usr/bin/zsh $USER

# Fazer logout/login
exit
```

### Problema: Plugins não funcionam

**Sintoma:**
```
Autosuggestions não aparecem
```

**Diagnóstico:**
```bash
# Verificar se plugins foram baixados
ls -la ~/.zsh-plugins/

# Verificar se estão sendo carregados
grep "source.*zsh-plugins" ~/.zshrc
```

**Solução:**
```bash
# Re-baixar plugins
rm -rf ~/.zsh-plugins
ansible-playbook -i inventory.yml site.yml --tags zsh

# Ou manualmente
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ~/.zsh-plugins/zsh-autosuggestions
```

### Problema: Keybindings não funcionam

**Sintoma:**
```
Ctrl+← não move palavra
```

**Causa:**
Terminal pode usar sequências diferentes.

**Solução:**
```bash
# Descobrir sequência do seu terminal
cat -v
# Pressione Ctrl+← e veja a sequência

# Adicionar ao .zshrc
bindkey '^[[1;5D' backward-word  # Ajustar sequência
```

### Problema: Prompt Git lento

**Sintoma:**
```
Prompt demora para aparecer em repositórios grandes
```

**Solução:**
```bash
# Simplificar função parse_git_info
# Editar ~/.zshrc

parse_git_info() {
  if git rev-parse --is-inside-work-tree &>/dev/null; then
    local branch="$(git symbolic-ref --short HEAD 2>/dev/null)"
    echo " ($branch)"
  fi
}
```

### Problema: Histórico não salva

**Sintoma:**
```
Comandos não aparecem no histórico
```

**Solução:**
```bash
# Verificar HISTFILE
echo $HISTFILE

# Verificar permissões
ls -la ~/.histfile

# Recriar arquivo
rm ~/.histfile
touch ~/.histfile
chmod 600 ~/.histfile
```

### Problema: Completion não funciona

**Sintoma:**
```
Tab não completa comandos
```

**Solução:**
```bash
# Recarregar completion
autoload -U compinit && compinit

# Ou adicionar ao .zshrc
rm -f ~/.zcompdump
autoload -U compinit && compinit
```

## Comandos Úteis do ZSH

### Navegação

```bash
# Mudar para diretório sem cd
/etc/nginx  # Em vez de: cd /etc/nginx

# Voltar ao diretório anterior
cd -

# Ir para home
cd ~
# Ou simplesmente
cd

# Stack de diretórios
pushd /tmp    # Adiciona ao stack e vai
popd          # Volta ao diretório anterior
dirs -v       # Lista stack
```

### Histórico

```bash
# Buscar no histórico
Ctrl+R        # Busca interativa

# Executar último comando
!!

# Executar comando N do histórico
!123

# Executar último comando que começa com 'git'
!git

# Substituir no último comando
^antigo^novo

# Ver histórico
history
history | grep git
```

### Globbing Avançado

```bash
# Todos os arquivos .txt recursivamente
**/*.txt

# Arquivos modificados hoje
*(m0)

# Arquivos maiores que 10MB
*(Lm+10)

# Diretórios vazios
*(/^F)

# Arquivos executáveis
*(*)
```

### Aliases Globais

```bash
# Adicionar ao .zshrc
alias -g G='| grep'
alias -g L='| less'
alias -g H='| head'
alias -g T='| tail'

# Usar
ps aux G firefox
dmesg L
ls -la H -20
```

## Frameworks Alternativos

Se você quiser usar frameworks prontos (não incluídos nesta role):

### Oh My Zsh

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### Prezto

```bash
git clone --recursive https://github.com/sorin-ionescu/prezto.git "${ZDOTDIR:-$HOME}/.zprezto"
```

### Zinit

```bash
bash -c "$(curl --fail --show-error --silent --location https://raw.githubusercontent.com/zdharma-continuum/zinit/HEAD/scripts/install.sh)"
```

**Nota:** Esta role fornece configuração manual e leve, sem frameworks pesados.

## Boas Práticas

### 1. Backup do .zshrc

```bash
# Antes de modificar
cp ~/.zshrc ~/.zshrc.backup

# Restaurar se necessário
cp ~/.zshrc.backup ~/.zshrc
```

### 2. Testar Mudanças

```bash
# Testar novo .zshrc sem aplicar
zsh -c 'source ~/.zshrc.new && zsh'

# Se funcionar, aplicar
mv ~/.zshrc.new ~/.zshrc
```

### 3. Atualizar Plugins

```bash
# Script de atualização
cat > ~/.local/bin/update-zsh-plugins << 'EOF'
#!/bin/bash
for plugin in ~/.zsh-plugins/*; do
  echo "Atualizando $(basename $plugin)..."
  git -C "$plugin" pull
done
EOF

chmod +x ~/.local/bin/update-zsh-plugins

# Executar
update-zsh-plugins
```

### 4. Perfil de Performance

```bash
# Medir tempo de inicialização
time zsh -i -c exit

# Profiling detalhado
zmodload zsh/zprof
# Adicionar ao início do .zshrc

# Ver relatório
zprof
# Adicionar ao final do .zshrc
```

## Recursos Adicionais

### Links Úteis

- **ZSH:** https://www.zsh.org/
- **Documentação:** https://zsh.sourceforge.io/Doc/
- **Plugins:** https://github.com/unixorn/awesome-zsh-plugins
- **Temas:** https://github.com/ohmyzsh/ohmyzsh/wiki/Themes

### Comunidades

- **r/zsh:** Reddit
- **ZSH Users:** https://www.zsh.org/mla/users/
- **Stack Overflow:** Tag `zsh`

## Comparação: ZSH vs. Bash

| Aspecto | ZSH | Bash |
|---------|-----|------|
| **Completion** | ✅ Avançado | ⚠️ Básico |
| **Plugins** | ✅ Muitos | ⚠️ Poucos |
| **Globbing** | ✅ Poderoso | ⚠️ Limitado |
| **Prompt** | ✅ Customizável | ⚠️ Básico |
| **Histórico** | ✅ Compartilhado | ⚠️ Por sessão |
| **Performance** | ✅ Rápido | ✅ Rápido |
| **Compatibilidade** | ⚠️ Quase POSIX | ✅ POSIX |
| **Padrão** | ⚠️ Não | ✅ Sim |

## Notas Importantes

1. **Compatibilidade:** ZSH é quase 100% compatível com Bash, mas há diferenças sutis.

2. **Scripts:** Use `#!/bin/bash` em scripts para portabilidade, não `#!/bin/zsh`.

3. **Performance:** ZSH pode ser ligeiramente mais lento que Bash em inicialização.

4. **Aprendizado:** Leva tempo para aproveitar todos os recursos do ZSH.

5. **Reversão:** Sempre pode voltar ao Bash: `chsh -s /bin/bash`

6. **Plugins:** Muitos plugins podem deixar o shell lento. Use apenas os necessários.

## Suporte

Para problemas ou dúvidas:
1. Consulte a seção de Troubleshooting acima
2. Documentação do ZSH: https://zsh.sourceforge.io/Doc/
3. ZSH Wiki: https://wiki.archlinux.org/title/Zsh
4. GitHub dos plugins
5. Abra uma issue no repositório do projeto

## Licença

Este código é fornecido como está, sem garantias. Use por sua conta e risco.