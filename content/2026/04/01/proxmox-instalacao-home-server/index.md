---
title: "Home Server - Instalação do Proxmox VE"
date: 2026-04-01T15:00:00-03:00
slug: "proxmox-instalacao-home-server"
tags:
  - homelab
  - home-server
  - proxmox
  - linux
  - virtualização
  - tutorial
draft: false
description: "Passo a passo completo da instalação do Proxmox VE 8 no home server: pendrive bootável, wizard de instalação, configuração de rede, repositórios e primeiros comandos."
---

No [post anterior](../../../03/02/home-server-por-que-montar-e-hardware-escolhido/) falei sobre os motivos que me levaram a montar um home server e detalhe cada peça do hardware escolhido. Com o hardware montado e na bancada, o próximo passo era colocar o sistema pra rodar. É aqui que o Proxmox entra.

Este post cobre a instalação do zero: da ISO no pendrive até o sistema atualizado e pronto para receber as primeiras VMs e containers.

---

## O que é o Proxmox VE

O **Proxmox Virtual Environment** é um hipervisor tipo 1 - isso significa que ele roda direto no hardware, sem um sistema operacional hospedeiro por baixo. É baseado em Debian Linux, é gratuito, open-source e tem interface web completa para gerenciamento.

A proposta central é simples: em vez de dedicar uma máquina física por serviço, você virtualiza tudo. Cada aplicação roda numa VM ou container isolado, compartilhando o mesmo hardware físico. O Proxmox orquestra isso por uma interface acessível por qualquer navegador, sem precisar de teclado e monitor permanentemente conectados ao servidor.

### Hipervisor tipo 1 vs tipo 2

Existem duas categorias de hipervisores:

- **Tipo 1 (bare metal):** roda direto no hardware. Proxmox, VMware ESXi e Microsoft Hyper-V são exemplos. Melhor performance, menor overhead.
- **Tipo 2 (hosted):** roda sobre um sistema operacional convencional. VirtualBox e VMware Workstation entram aqui. Mais fácil de instalar, mas com mais camadas entre o hardware e a VM.

Para um servidor que vai ficar ligado 24/7 e rodar serviços reais, tipo 1 é o caminho.

### VMs vs Containers no Proxmox

O Proxmox suporta dois modelos de virtualização:

```
VM (KVM/QEMU)
├── Sistema operacional completo e independente
├── CPU e memória dedicados (ou compartilhados com limites)
├── Pode rodar qualquer OS: Windows, Linux, BSD
└── Isolamento total do host

Container (LXC)
├── Compartilha o kernel do host
├── Apenas Linux
├── Muito mais leve em memória e CPU
└── Inicialização quase instantânea
```

A escolha entre um e outro depende do workload. Windows obrigatoriamente precisa de VM. Serviços Linux leves - Pi-hole, Nginx Proxy Manager, Uptime Kuma, rodam perfeitamente em containers LXC com uma fração dos recursos que uma VM consumiria.

### Por que Proxmox e não as outras opções

No post anterior já citei as alternativas que analisei. Para completar o contexto:

- **VMware ESXi:** ficou pago após aquisição pela Broadcom. Fora de cogitação para homelab.
- **Microsoft Hyper-V:** amarrado ao ecossistema Windows Server. Não era o que eu queria.
- **TrueNAS Scale:** excelente para NAS, mas o foco em storage não era o que eu precisava como plataforma principal.
- **Proxmox:** gratuito, open-source, usado em produção corporativa, suporta ZFS nativamente, interface web completa. Caixa fechada.

---

## Hardware utilizado nesta instalação

Apenas para contextualizar o que foi usado e servir de referência para quem tem configuração parecida:

| Componente | Modelo |
|---|---|
| Placa-mãe / CPU | Topton N5105 - 4 cores, TDP 10W |
| SSD sistema | Kingston NV2 1TB NVMe (`/dev/nvme0n1`) |
| SSD projetos | MOVESPEED 1TB PCIe 4.0 (`/dev/nvme1n1`) |
| HDs de dados | 2x WD Red Plus 4TB (`/dev/sda`, `/dev/sdb`) |
| Memória | 2x Corsair Vengeance 16GB DDR4 - 32GB total |
| Fonte | Cooler Master G500 Gold 500W |
| NICs | 4x Intel i226-V 2.5G (integradas na Topton) |

---

## O que você vai precisar

Antes de começar, separe:

- **Pendrive** de pelo menos 8 GB (o conteúdo será apagado)
- **Cabo de rede** conectado ao roteador, a instalação configura a rede e você vai precisar de acesso logo depois
- **Monitor e teclado** temporariamente conectados ao servidor para o processo de instalação
- Um computador separado para criar o pendrive bootável e, depois, para acessar a interface web

---

## Parte 1 - Pendrive bootável

### Download da ISO

Acesse **proxmox.com/downloads**, clique em **Proxmox Virtual Environment** e baixe a ISO mais recente. O arquivo terá um nome no formato `proxmox-ve_8.x-x.iso`.

> Sempre baixe a versão mais recente disponível. O processo de instalação descrito aqui segue o wizard do Proxmox VE 8, que é o que está em uso no servidor.

### Gravando com Balena Etcher

O **Balena Etcher** é a ferramenta mais direta para isso. Gratuito, roda em Windows, macOS e Linux, e valida a gravação automaticamente. Baixe em **balena.io/etcher**.

Com o Etcher aberto:

1. **Flash from file** → selecione o arquivo `.iso` baixado
2. **Select target** → selecione o pendrive. Confira bem qual é o correto se tiver mais de um conectado
3. **Flash!** → aguarde. O processo leva entre 2 e 5 minutos e o Etcher valida a gravação ao final

> Se estiver no Windows, é normal o sistema exibir uma mensagem pedindo para formatar o pendrive após a gravação, ele não reconhece o sistema de arquivos Linux. **Não formate.** Cancele a mensagem e o pendrive está pronto.

**Alternativas ao Etcher:** Rufus (Windows), Ventoy, ou `dd` no Linux/macOS.

```bash
# Gravar com dd no Linux (substitua /dev/sdX pelo seu pendrive)
dd if=proxmox-ve_8.x-x.iso of=/dev/sdX bs=4M status=progress && sync
```

---

## Parte 2 - Instalação

### Configurando o boot pela BIOS

Conecte o pendrive ao servidor, ligue a máquina e acesse a BIOS (geralmente `DEL`, `F2` ou `F10` - aqui depende da placa-mãe). Na Topton N5105 é `DEL`.

O que ajustar:

1. **Virtualização habilitada** - procure por `Intel VT-x` ou `Virtualization Technology`. Precisa estar `Enabled`. Sem isso, o Proxmox não consegue criar VMs.
2. **Boot order** - coloque o USB em primeiro lugar. Alternativa: usar o Boot Menu (`F11` ou `F12`) para boot único do pendrive sem mexer na ordem permanente.
3. **Boot Mode** - deixe em **UEFI**. O Proxmox suporta UEFI e é o modo recomendado.

Salve com `F10` e o servidor vai reiniciar carregando pelo pendrive.

### Tela inicial do instalador

Ao carregar, aparece o menu do instalador do Proxmox. Selecione **Install Proxmox VE (Graphical)** e aguarde o ambiente gráfico do wizard carregar.

### Tela 1 - EULA

Leia os termos e clique em **I agree** para continuar.

### Tela 2 - Disco de destino

Aqui você escolhe onde o Proxmox será instalado e o sistema de arquivos.

**Target Harddisk:** selecione o disco de destino. No meu caso, o **Kingston NV2 1TB** (`/dev/nvme0n1`).

Clique em **Options** para abrir as opções de filesystem.

**Filesystem escolhido: ZFS (RAID0)**

Sim, ZFS num disco único. A razão é que o Proxmox se beneficia bastante dos recursos do ZFS mesmo sem RAID: checksums de integridade em todos os blocos de dados, snapshots nativos (úteis para voltar o estado do sistema antes de uma atualização problemática) e compressão transparente.

```
ext4    → simples, confiável, sem overhead
ZFS     → checksums, snapshots, compressão - mais recursos, mais RAM
```

Com 32GB de RAM na máquina, o overhead de memória do ZFS num disco único é irrelevante. A escolha valeu.

Deixe as demais opções do ZFS no padrão (`ashift=12`, compressão `on`) e clique em **Next**.

### Tela 3 - Localização e teclado

- **Country:** Brazil
- **Timezone:** America/Sao_Paulo
- **Keyboard Layout:** Brazilian Portuguese (ou `br-abnt2`, dependendo do seu teclado)

Clique em **Next**.

### Tela 4 - Senha e e-mail

**Password:** defina a senha do usuário `root`. Essa senha será usada tanto para o login na interface web quanto para acesso SSH.

> Use uma senha forte. Anote em algum lugar seguro - gerenciador de senhas, como exemplo: **Bitwarden**, papel numa gaveta trancada, o que for. Perder o acesso root a um servidor sem KVM é uma dor de cabeça desnecessária.

**E-mail:** um endereço para notificações do sistema. O Proxmox envia alertas de storage, backups com falha e atualizações disponíveis por aqui.

Clique em **Next**.

### Tela 5 - Configuração de rede

Esta é a tela mais importante da instalação. O que você definir aqui determina como você vai acessar o servidor depois.

**Management Interface:** selecione a placa de rede para gerenciamento. A Topton N5105 tem quatro NICs Intel i226-V, que aparecem como `enp4s0`, `enp5s0`, `enp6s0` e `enp7s0`. Usei `enp4s0`, a primeira porta.

As configurações que defini:

```
Hostname (FQDN): berserk.local
IP Address:      192.168.1.10/24
Gateway:         192.168.1.1
DNS Server:      1.1.1.1
```

Adapte o IP e gateway para a faixa da sua rede. O importante é que o IP seja fixo e que esteja dentro da faixa do seu roteador.

> Anote essas informações antes de clicar em Next. Você vai precisar do IP para acessar a interface web logo depois da instalação.

Clique em **Next**.

### Tela 6 - Confirmação

O wizard exibe um resumo de tudo. Confira:

- Disco correto selecionado
- Filesystem correto (ZFS no meu caso)
- IP e hostname corretos
- Senha anotada

Clique em **Install**.

### Processo de instalação

O instalador formata o disco, instala o Debian base, instala o Proxmox VE por cima e configura o bootloader. Em SSD NVMe leva entre 2 e 3 minutos.

Ao finalizar, aparece:

```
Installation successful!
Please remove the installation medium and press Enter to reboot.
```

Remova o pendrive e pressione Enter. O servidor vai reiniciar carregando o Proxmox do SSD.

---

## Parte 3 - Primeiro acesso à interface web

Após o boot, o servidor exibe no terminal um endereço de acesso:

```
Please use your web browser to configure this server - connect to:
https://192.168.1.10:8006/
```

No navegador do seu computador, acesse:

```
https://192.168.1.10:8006
```

### Aviso de certificado

O navegador vai exibir um aviso de segurança - algo como "Sua conexão não é privada" ou "This connection is not private". Isso é esperado: o Proxmox usa um certificado TLS auto-assinado por padrão, e o navegador não o reconhece como confiável.

Clique em **Avançado** → **Continuar para 192.168.1.10**. Em uma rede local com servidor que você controla, isso é seguro.

### Login

```
Username: root
Password: [senha definida na instalação]
Realm:    Linux PAM standard authentication
```

Clique em **Login**.

### Pop-up "No valid subscription"

Na primeira entrada - e em todos os logins enquanto você não tiver uma assinatura paga - o Proxmox exibe esse aviso. Ele informa que você está usando a versão sem suporte comercial.

As funcionalidades são idênticas entre a versão gratuita e a paga. A assinatura cobre apenas suporte técnico profissional da Proxmox GmbH. Clique em **OK** e siga em frente.

---

## Parte 4 - Configuração inicial

### Repositórios: desabilitar Enterprise, habilitar No-Subscription

Por padrão, o Proxmox aponta para o repositório `enterprise.proxmox.com`, que exige assinatura ativa. Sem ela, qualquer tentativa de atualizar o sistema retorna erro de autenticação.

A solução é desabilitar o repositório Enterprise e habilitar o No-Subscription, que é o repositório gratuito com as mesmas versões de pacotes.

Navegue até:

```
Datacenter → berserk → Updates → Repositories
```
> **No lugar de berserk, aqui escolha o seu node.**

O que fazer:

1. Localize a linha com `enterprise.proxmox.com` - clique nela e clique em **Disable**
2. Clique em **Add** → selecione **No-Subscription** → **Add**

Resultado esperado:

```
https://enterprise.proxmox.com/debian/pve   → Disabled
http://download.proxmox.com/debian/pve      → Enabled (No-Subscription)
```

> Existe um terceiro repositório chamado `pvetest`, que contém versões em fase de testes. **Não habilite** em ambiente que você quer estável. No-Subscription já cobre tudo que você precisa.

### Atualizar o sistema

Com os repositórios corretos configurados, vá até:

```
Datacenter → berserk → Updates
```
> **No lugar de berserk, aqui escolha o seu node.**

Clique em **Refresh** para atualizar a lista de pacotes disponíveis. Aguarde o processo terminar.

Depois clique em **Upgrade**. Um terminal abre listando os pacotes que serão atualizados. Confirme com `Y` + Enter e aguarde o download e instalação.

```bash
# O mesmo pode ser feito via SSH
apt update && apt dist-upgrade -y
```

Após terminar, reinicie o servidor se o upgrade incluir um novo kernel (o terminal vai avisar):

```bash
reboot
```

---

## Parte 5 - Dashboard e o que monitorar

Após o login, vá em:

```
Datacenter → berserk → Summary
```
> **No lugar de berserk, aqui escolha o seu node.**

O dashboard exibe o estado atual do servidor. No meu:

```
CPU:     4x Intel Celeron N5105 @ 2.00GHz
RAM:     9.17 GiB em uso / 31.19 GiB total
HD:      69.18 GiB em uso / 93.93 GiB (volume ZFS do sistema)
Kernel:  Linux 6.8.12-20-pve
Versão:  pve-manager/8.4.17
```

### Pools de storage

Em **Datacenter → berserk → Disks → ZFS** e **Storage**, você vai ver os volumes que o instalador criou e os que você adicionar depois. No meu setup ficou assim:

| Pool | Tipo | Uso |
|---|---|---|
| `local (berserk)` | Directory | ISOs, backups, templates de containers |
| `local-lvm (berserk)` | LVM-Thin | Disk images, containers |
| `local-zfs (berserk)` | ZFS | Disk images, containers (Kingston NV2) |
| `zpool (berserk)` | ZFS | Disk images, containers (WD Red Plus Mirror) |

O `zpool` é o ZFS Mirror dos dois WD Red Plus 4TB - os dados replicados entre os dois discos ficam aqui.

### Interfaces de rede

Em **Datacenter → berserk → Network**:

```
enp4s0  → NIC física de gerenciamento
enp5s0  → NIC física (disponível para VMs)
enp6s0  → NIC física (disponível para VMs)
enp7s0  → NIC física (disponível para VMs)
vmbr0   → Linux Bridge em cima de enp4s0 (gerenciamento + VMs)
```

A bridge `vmbr0` é o que permite que as VMs se comuniquem com a rede física. Quando criar uma VM, você vai atribuir `vmbr0` como interface de rede dela.

---

## Comandos úteis via SSH

Com o Proxmox rodando, você pode gerenciar muita coisa pelo terminal via SSH:

```bash
# Conectar ao servidor
ssh root@192.168.1.10

# Versão do Proxmox
pveversion

# Status dos serviços principais
systemctl status pve-cluster pveproxy pvedaemon

# Atualizar sistema
apt update && apt dist-upgrade -y

# Listar VMs
qm list

# Listar containers LXC
pct list

# Status dos pools ZFS
zpool status

# Verificar integridade dos dados (ZFS scrub)
zpool scrub zpool

# Espaço em disco por pool
zfs list
```

---

## Troubleshooting - Quando as coisas não saem como esperado

### Interface web não abre

Se `https://192.168.1.10:8006` não carregar:

```bash
# Verificar se o serviço está rodando
systemctl status pveproxy

# Reiniciar o serviço
systemctl restart pveproxy

# Confirmar que a porta está escutando
ss -tlnp | grep 8006
```

Também verifique se o IP configurado na instalação está correto. Se errou o IP durante o wizard, edite o arquivo de configuração de rede:

```bash
nano /etc/network/interfaces
# Ajuste o endereço em iface vmbr0
systemctl restart networking
```

### Erro ao atualizar - repositório Enterprise

```
E: Failed to fetch https://enterprise.proxmox.com/...
   401 Unauthorized
```

Causa: repositório Enterprise ativo sem assinatura. Siga os passos da Parte 4 para desabilitar o Enterprise e habilitar o No-Subscription.

Via terminal também funciona:

```bash
# Desabilitar repo Enterprise
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Adicionar repo No-Subscription
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update
```

### VM não inicia

Verifique os pontos mais comuns:

- RAM disponível: se todas as VMs somadas excedem a memória física, o Proxmox recusa iniciar mais
- Storage com espaço: o disco de destino da VM precisa ter espaço livre
- Virtualização habilitada na BIOS: se o Intel VT-x não estiver ativo, VMs KVM não vão funcionar

```bash
# Ver logs de erro de uma VM específica (substitua 100 pelo ID da VM)
journalctl -u pve-cluster | tail -50
qm showcmd 100
```

### VM sem acesso à rede

Verifique se a bridge `vmbr0` está corretamente configurada:

```bash
ip link show vmbr0
bridge link show
```

Na interface web, confirme que a VM tem uma placa de rede atribuída e que ela aponta para `vmbr0`. O driver recomendado é **VirtIO** - melhor performance que o emulado `e1000`.

---

## Próximos passos

O Proxmox está instalado, atualizado e com os discos reconhecidos. O que vem a seguir na série:

- **Configuração do ZFS Mirror** com os WD Red Plus - criação do pool, scrub programado e snapshots automáticos
- **Primeira VM** - subindo um Ubuntu Server para começar a rodar serviços
- **Containers LXC** - serviços leves sem o overhead de uma VM completa
- **Jellyfin** - servidor de mídia rodando sobre o Proxmox

O ambiente está pronto. Agora é hora de configurar e usar.
