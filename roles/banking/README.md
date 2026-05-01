# Role: Banking

Esta role configura o ambiente para aplicações bancárias, incluindo a instalação do Warsaw Banking Security no Distrobox.

## Descrição

A role `banking` cria um container Distrobox baseado no openSUSE Leap e instala o Warsaw Banking Security, necessário para acessar sites de bancos brasileiros.

## Requisitos

- Distrobox instalado e configurado
- Podman ou Docker
- Acesso à internet para baixar o instalador do Warsaw
- Permissões de superusuário (a role usa `become: true`)

## Variáveis

Definidas em `defaults/main.yml`:

```yaml
banking_box_name: "banking-workspace"
warsaw_url: "https://guardiao.itau.com.br/warsaw/warsaw_opensuse.run"
warsaw_rpm_name: "warsaw-2.21.9-2.x86_64.rpm"
```

### Variáveis Globais Necessárias

Definidas em `group_vars/all.yml`:

```yaml
leap_image: "leap:latest"  # Imagem do openSUSE Leap para o container
```

## Estrutura

```
roles/banking/
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

### 1. Criação do Container

A role cria um container Distrobox chamado `banking-workspace` baseado no openSUSE Leap:

```bash
distrobox create -n banking-workspace --image leap:latest --yes
```

### 2. Instalação do Warsaw

O processo de instalação do Warsaw foi otimizado para ser **não-interativo** e **robusto**:

#### a) Verificação de Instalação Prévia
- Verifica se o Warsaw já está instalado no container
- Pula a instalação se já estiver presente

#### b) Instalação de Dependências
```bash
distrobox enter banking-workspace -- sudo zypper -n in \
  libXcursor1 libXrender1 libXext6 libXfixes3 libXft2 \
  iproute2 tar wget
```

#### c) Download do Instalador
- Baixa o `warsaw_opensuse.run` do site oficial do Itaú
- Salva em `/tmp/warsaw_opensuse.run` com permissões de execução

#### d) Extração Não-Interativa
```bash
export DISPLAY=
export DEBIAN_FRONTEND=noninteractive
./warsaw_opensuse.run --target /tmp/warsaw_extract --noexec
```

**Flags importantes:**
- `--target /tmp/warsaw_extract`: Especifica diretório de saída
- `--noexec`: Evita execução automática após extração
- `DISPLAY=`: Desabilita tentativas de abrir GUI
- `DEBIAN_FRONTEND=noninteractive`: Modo não-interativo

#### e) Localização do RPM
Busca o RPM extraído em múltiplos locais:
1. `/tmp/warsaw_extract/*.rpm`
2. `/tmp/*.rpm`
3. Busca recursiva em `/tmp`

#### f) Instalação do RPM
```bash
distrobox enter banking-workspace -- sudo zypper -n \
  --no-gpg-checks in --allow-unsigned-rpm <caminho-do-rpm>
```

#### g) Verificação do Binário Core
Localiza o binário `core` do Warsaw em:
1. `/usr/local/bin/warsaw/core`
2. `/usr/bin/warsaw/core`
3. `/opt/warsaw/core`
4. Busca genérica em `/usr` e `/opt`

### 3. Serviço Systemd

A role cria um serviço systemd em `/etc/systemd/system/warsaw-container.service`:

```ini
[Unit]
Description=Servico Warsaw Banking no Distrobox
After=network-online.target
Wants=network-online.target

[Service]
User=<seu-usuario>
Delegate=yes
# Type=oneshot com RemainAfterExit garante que o systemd
# considere o serviço ativo mesmo após o comando inicial fechar.
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/distrobox enter banking-workspace -- sudo <path-detectado>/core
# O Warsaw costuma demorar uns segundos para subir o socket 30900
ExecStartPost=/usr/bin/sleep 5
# Reiniciar apenas em caso de falha
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Características:**
- Path do binário `core` é **detectado dinamicamente**
- Serviço roda como seu usuário (não como root)
- Inicia automaticamente no boot
- **Type=oneshot + RemainAfterExit=yes**: Evita loop de reinicialização infinita
  - O Warsaw daemoniza (roda em segundo plano) após ser iniciado
  - O systemd considera o serviço ativo mesmo após o comando inicial terminar
  - Sem essa configuração, o systemd tentaria reiniciar o serviço continuamente
- **ExecStartPost com sleep 5**: Aguarda o socket 30900 estar pronto
- **Restart=on-failure**: Reinicia apenas se houver falha real

### 4. Verificações e Logs

A role inclui verificações em cada etapa:
- ✅ RPM extraído com sucesso
- ✅ Warsaw instalado corretamente
- ✅ Binário `core` existe e é executável
- ✅ Serviço systemd criado e iniciado
- ✅ Status do serviço verificado

## Uso

### Executar a Role

```bash
ansible-playbook -i inventory.yml site.yml --tags banking
```

### Verificar Status do Warsaw

```bash
# Status do serviço
sudo systemctl status warsaw-container.service

# Logs do serviço
sudo journalctl -u warsaw-container.service -n 50

# Verificar se o Warsaw está rodando no container
distrobox enter banking-workspace -- ps aux | grep warsaw
```

### Acessar o Container

```bash
# Entrar no container
distrobox enter banking-workspace

# Verificar instalação do Warsaw
rpm -qa | grep warsaw

# Verificar binário core
ls -la /usr/local/bin/warsaw/core
```

### Testar o Warsaw

Após a instalação, acesse:
```
https://127.0.0.1:30900/
```

Você deve ver a interface web do Warsaw Banking Security.

## Troubleshooting

### Problema: Loop de Reinicialização Infinita

**Sintoma:**
```bash
$ sudo systemctl status warsaw-container.service
● warsaw-container.service - Servico Warsaw Banking no Distrobox
   Active: activating (auto-restart) (Result: exit-code)
   # Contador de reinícios aumentando continuamente
```

**Causa:**
O Warsaw daemoniza (roda em segundo plano) após ser iniciado. Quando o comando `distrobox enter` termina com sucesso (exit code 0), o systemd interpreta isso como "o serviço morreu" e tenta reiniciá-lo, criando um loop infinito.

**Solução:**
A configuração atual já resolve este problema usando:
- `Type=oneshot`: Indica que o comando é executado uma vez
- `RemainAfterExit=yes`: Mantém o serviço como "active" mesmo após o comando terminar
- `Restart=on-failure`: Reinicia apenas em caso de falha real

Se você ainda estiver enfrentando este problema com uma versão antiga da role:

```bash
# 1. Parar o serviço
sudo systemctl stop warsaw-container.service

# 2. Editar o arquivo de serviço
sudo nano /etc/systemd/system/warsaw-container.service

# 3. Garantir que tenha estas linhas na seção [Service]:
Type=oneshot
RemainAfterExit=yes
ExecStartPost=/usr/bin/sleep 5
Restart=on-failure
RestartSec=10

# 4. Recarregar e reiniciar
sudo systemctl daemon-reload
sudo systemctl start warsaw-container.service

# 5. Verificar status (deve mostrar "active (exited)")
sudo systemctl status warsaw-container.service

# 6. Verificar socket do Warsaw
ss -tulpn | grep 30900
```

### Problema: Serviço não inicia

**Sintoma:**
```bash
$ sudo systemctl status warsaw-container.service
● warsaw-container.service - Servico Warsaw Banking no Distrobox
   Active: failed (Result: exit-code)
```

**Diagnóstico:**
```bash
# Ver logs detalhados
sudo journalctl -u warsaw-container.service -n 100 --no-pager

# Verificar se o container existe
distrobox list | grep banking-workspace

# Verificar se o Warsaw está instalado
distrobox enter banking-workspace -- rpm -qa | grep warsaw

# Verificar binário core
distrobox enter banking-workspace -- find /usr /opt -name core -path '*/warsaw/*'
```

**Soluções:**

1. **Container não existe:**
   ```bash
   distrobox create -n banking-workspace --image leap:latest --yes
   ```

2. **Warsaw não instalado:**
   ```bash
   # Re-executar a role
   ansible-playbook -i inventory.yml site.yml --tags banking
   ```

3. **Binário core não encontrado:**
   ```bash
   # Verificar instalação manual
   distrobox enter banking-workspace
   sudo zypper in --allow-unsigned-rpm /tmp/warsaw*.rpm
   ```

### Problema: Extração do warsaw_opensuse.run falha

**Sintoma:**
```
ERRO: RPM do Warsaw não foi encontrado após extração!
```

**Causas possíveis:**
1. Instalador mudou de formato
2. Problemas de rede durante download
3. Permissões insuficientes em `/tmp`

**Soluções:**

1. **Verificar download:**
   ```bash
   ls -lh /tmp/warsaw_opensuse.run
   file /tmp/warsaw_opensuse.run
   ```

2. **Tentar extração manual:**
   ```bash
   distrobox enter banking-workspace
   cd /tmp
   chmod +x warsaw_opensuse.run
   ./warsaw_opensuse.run --target /tmp/warsaw_extract --noexec
   ls -la /tmp/warsaw_extract/
   ```

3. **Verificar logs de extração:**
   ```bash
   cat /tmp/warsaw_extract.log
   ```

### Problema: Warsaw instalado mas serviço falha

**Sintoma:**
```
sudo: /usr/local/bin/warsaw/core: comando não encontrado
```

**Diagnóstico:**
```bash
# Encontrar onde o Warsaw realmente instalou
distrobox enter banking-workspace -- find / -name core -path '*/warsaw/*' 2>/dev/null
```

**Solução:**
A role agora detecta automaticamente o path correto. Se ainda assim falhar:

1. **Verificar path manualmente:**
   ```bash
   distrobox enter banking-workspace
   rpm -ql $(rpm -qa | grep warsaw) | grep core
   ```

2. **Atualizar serviço manualmente:**
   ```bash
   sudo nano /etc/systemd/system/warsaw-container.service
   # Atualizar linha ExecStart com o path correto
   sudo systemctl daemon-reload
   sudo systemctl restart warsaw-container.service
   ```

### Problema: Versão do Warsaw mudou

**Sintoma:**
O instalador baixa uma versão diferente do RPM.

**Solução:**
A role agora busca qualquer arquivo `warsaw*.rpm`, independente da versão. Não é necessário atualizar a variável `warsaw_rpm_name`.

### Problema: Permissões negadas

**Sintoma:**
```
Permission denied ao executar warsaw_opensuse.run
```

**Solução:**
```bash
# Garantir permissões corretas
chmod +x /tmp/warsaw_opensuse.run

# Verificar se o usuário tem acesso ao /tmp
ls -ld /tmp
```

## Limpeza

### Remover o Warsaw

```bash
# Parar serviço
sudo systemctl stop warsaw-container.service
sudo systemctl disable warsaw-container.service

# Remover serviço
sudo rm /etc/systemd/system/warsaw-container.service
sudo systemctl daemon-reload

# Remover Warsaw do container
distrobox enter banking-workspace -- sudo zypper rm warsaw

# Ou remover o container inteiro
distrobox rm banking-workspace
```

### Limpar arquivos temporários

```bash
rm -f /tmp/warsaw_opensuse.run
rm -rf /tmp/warsaw_extract
rm -f /tmp/warsaw_extract.log
```

## Melhorias Implementadas

### Versão Anterior vs. Nova

| Aspecto | Antes | Agora |
|---------|-------|-------|
| **Extração** | Timeout de 15s com `\|\| true` | Extração não-interativa com `--noexec` |
| **Verificação RPM** | Busca simples | Busca em múltiplos locais |
| **Tratamento de erros** | Erros ignorados | Falha explícita com mensagens claras |
| **Path do core** | Hardcoded `/usr/local/bin/warsaw/core` | Detecção dinâmica |
| **Logs** | Mínimos | Detalhados em cada etapa |
| **Idempotência** | Parcial | Completa (verifica instalação prévia) |
| **Versão do RPM** | Dependente de variável | Independente de versão |

## Notas Importantes

1. **Ansible com Elevação:** A role usa `become: true` para tarefas que requerem privilégios de superusuário (criação do serviço systemd).

2. **Container Rootless:** O container Distrobox roda como seu usuário, não como root.

3. **Persistência:** O serviço systemd garante que o Warsaw inicie automaticamente no boot.

4. **Segurança:** O Warsaw é instalado apenas dentro do container, isolado do sistema host.

5. **Compatibilidade:** Testado com openSUSE Leap no container. Pode funcionar com outras distribuições com ajustes.

## Suporte

Para problemas ou dúvidas:
1. Verifique os logs: `sudo journalctl -u warsaw-container.service`
2. Consulte a seção de Troubleshooting acima
3. Verifique a documentação oficial do Warsaw
4. Abra uma issue no repositório do projeto

## Licença

Este código é fornecido como está, sem garantias. Use por sua conta e risco.