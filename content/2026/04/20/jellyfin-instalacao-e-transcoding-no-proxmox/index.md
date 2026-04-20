---
title: "Home Server - Jellyfin: instalação e transcodificação de hardware no Proxmox"
date: 2026-04-20T09:00:00-03:00
slug: "jellyfin-instalacao-e-transcoding-no-proxmox"
tags:
  - homelab
  - home-server
  - proxmox
  - jellyfin
  - linux
  - tutorial
  - streaming
draft: false
description: "Como instalar o Jellyfin em um container LXC no Proxmox, configurar armazenamento ZFS, organizar bibliotecas de mídia e habilitar transcodificação de hardware com Intel Quick Sync no N5105."
---

No [post anterior](../12/proxmox-configuracao-inicial-home-server/) deixei o Proxmox configurado: discos reconhecidos, ZFS pools criados, backups agendados, IOMMU habilitado. O ambiente está pronto para receber serviços. O primeiro deles é o Jellyfin.

Este post cobre todo o processo: da criação do container LXC até a transcodificação de hardware funcionando com Intel Quick Sync. É o tutorial mais longo da série até agora, tem bastante coisa para configurar, mas o resultado vale a pena.

---

## O que é o Jellyfin

Jellyfin é um servidor de mídia open-source e gratuito. Ele permite organizar e fazer streaming de filmes, séries, animes e músicas para qualquer dispositivo na rede local: Smart TV, tablet, celular, navegador.

Funciona 100% local, sem nuvem, sem conta, sem assinatura. Você é dono do servidor e dos dados.

Se você já conhece Plex ou Emby, o Jellyfin é a alternativa livre:

| | Jellyfin | Plex | Emby |
|---|---|---|---|
| Preço | Gratuito | Freemium | Pago |
| Código | Open-source | Proprietário | Proprietário |
| Limitações | Nenhuma | Recursos pagos | Recursos pagos |
| Privacidade | Total | Coleta dados | Coleta dados |

Escolhi Jellyfin por ser completamente livre e por não depender de servidores externos para funcionar.

---

## Por que transcodificação de hardware importa

Diferentes dispositivos suportam diferentes formatos de vídeo. Um iPad reproduz quase tudo. Uma Smart TV antiga pode engasgar com HEVC. Quando o dispositivo não suporta o formato original, o Jellyfin precisa converter o vídeo em tempo real - isso é transcodificação.

> O **HEVC/H.265** é o formato de vídeo que oferece melhor qualidade com arquivo menor, mas exige mais do hardware para decodificar.

Sem aceleração de hardware, a CPU faz todo o trabalho. Num cenário de transcodificação pesada, o uso de CPU pode passar de **90%**. Com a GPU fazendo esse trabalho, o uso cai para **10-15%**.

O N5105 (Jasper Lake, Gen 11) tem GPU Intel integrada com Quick Sync Video. É exatamente o que preciso.

---

## Parte 1 - Criar o container LXC

### Baixar o template

No Proxmox, acesse **Storage (local) → CT Templates → Templates**. Procure por `ubuntu-24.04` ou `debian-12` e faça o download.

### Criar container privilegiado

Vamos usar container **privilegiado** para permitir acesso direto à GPU. Em **Create CT**:

**General:**
- **CT ID**: 102
- **Hostname**: jellyfin
- **Unprivileged container**: ❌ **desmarcar**
- **Password**: defina uma senha root

**Template:**
- **Storage**: local
- **Template**: ubuntu-24.04 (ou debian-12)

**Disks:**
- **Storage**: local-zfs
- **Disk size**: 40 GB

**CPU:**
- **Cores**: 4

**Memory:**
- **Memory**: 8192 MiB (8 GB)
- **Swap**: 512 MiB

**Network:**
- **Bridge**: vmbr0
- **IPv4**: DHCP ou estático (ex: `192.168.1.102/24`)
- **Gateway**: `192.168.1.1` (se estático)

**DNS:** Use host settings.

Clique em **Finish**, mas **não inicie** o container ainda.

---

## Parte 2 - Instalar o Jellyfin

### Iniciar e acessar o container

Selecione o container → **Start** → acesse via **Console** ou SSH.

### Instalar via repositório oficial

Estou usando **Ubuntu 24.04 LTS**. O método varia por distro:

**Ubuntu 24.04:**

```bash
# Container privilegiado roda como root - sudo é opcional,
# mas incluído aqui por boa prática e reprodutibilidade fora do container

# Baixar script oficial do Jellyfin
curl -s https://repo.jellyfin.org/install-debuntu.sh -O

# Verificar checksum antes de executar (recomendado)
# Hash atual em: https://repo.jellyfin.org/install-debuntu.sh.sha256sum
sha256sum install-debuntu.sh

# Executar
sudo bash install-debuntu.sh
```

**Debian 12** (extrepo, não disponível no Ubuntu):

```bash
sudo apt update && sudo apt install -y extrepo
sudo extrepo enable jellyfin
sudo apt update
sudo apt install -y jellyfin
```

Três pacotes são instalados: `jellyfin` (servidor), `jellyfin-ffmpeg` (transcodificação de vídeo) e `jellyfin-web` (interface web).

### Verificar se está rodando

```bash
systemctl status jellyfin
```

O status deve ser `active (running)`. Se não estiver:

```bash
systemctl start jellyfin
systemctl enable jellyfin
```

### Configuração inicial via web

Acesse `http://IP-DO-CONTAINER:8096` no navegador.

O Jellyfin apresenta um wizard de configuração inicial:

1. **Idioma** - selecione Português ou Inglês
2. **Criar conta admin** - defina o usuário e uma senha forte (essa é a conta principal)
3. **Bibliotecas** - pule por enquanto, vamos configurar depois
4. **Idioma de metadados** - Português (Brazil) ou English
5. **Acesso remoto** - marque "Allow remote connections" e "Enable automatic port mapping"
6. **Finalizar** - faça login com a conta criada

**Sobre o acesso remoto:**

- **Allow remote connections** - permite acessar o Jellyfin de fora da rede local (outro celular, casa de amigo, etc.)
- **Enable automatic port mapping (UPnP)** - o Jellyfin tenta configurar o roteador automaticamente para abrir a porta necessária

> ⚠️ UPnP pode não funcionar em roteadores antigos ou com UPnP desabilitado. Se o acesso externo não funcionar, configure **port forwarding** manualmente no seu roteador: encaminhe a porta **8096 (TCP)** para o IP do container.

Neste ponto o Jellyfin está rodando, mas sem conteúdo ainda.

---

## Parte 3 - Configurar armazenamento ZFS

O armazenamento de mídia fica no pool `zpool` (2x WD Red Plus 4TB em ZFS mirror), o mesmo que configurei nos posts anteriores. Vamos criar um dataset dedicado para o Jellyfin.

> No ZFS, um dataset é como uma pasta inteligente dentro do pool, tem configurações próprias de quota, compressão e permissões, e pode ser montado em qualquer lugar do sistema de forma independente.

### Criar dataset no host Proxmox

```bash
# Criar estrutura hierárquica
zfs create zpool/jellyfin
zfs create zpool/jellyfin/media

# Definir quota (opcional - limita quanto espaço o dataset pode usar)
# Obs: fica a critério de cada um, eu deixei livre
zfs set quota=1T zpool/jellyfin/media

# Verificar
zfs list | grep jellyfin
```

### Permissões do dataset

```bash
chmod -R 777 /zpool/jellyfin/media
```

> **Nota sobre segurança:** `777` é permissão total - adequado para um home server pessoal. Em ambientes compartilhados, configure permissões por usuário/grupo.

### Montar o dataset no container

Ainda no host Proxmox:

```bash
pct set 102 --mp2 /zpool/jellyfin/media,mp=/home/berserk/media
```

Parâmetros:
- `102` - ID do container
- `--mp2` - mount point 2 (use mp1, mp2, etc. conforme disponibilidade)
- `/zpool/jellyfin/media` - caminho no host
- `/home/berserk/media` - caminho dentro do container

### Criar pastas de mídia

Dentro do container:

```bash
mkdir -p /home/berserk/media/filmes
mkdir -p /home/berserk/media/series
mkdir -p /home/berserk/media/animes
mkdir -p /home/berserk/media/shows
```

A vantagem dessa estrutura é que o dataset ZFS pode ser compartilhado com outros containers futuros (Nextcloud, Samba, etc.), já que os dados ficam no host.

---

## Parte 4 - Organizar os arquivos de mídia

Antes de criar bibliotecas no Jellyfin, vale garantir que os arquivos estejam organizados corretamente. A nomenclatura influencia diretamente na busca automática de metadados.

### O que são metadados

Metadados são as informações que o Jellyfin busca automaticamente na internet para deixar sua biblioteca com cara de streaming profissional:

- **Títulos** - nome oficial do filme ou série
- **Sinopses e descrições** - resumo do enredo
- **Pôsteres e capas** - imagens de capa, backdrops e thumbnails dos episódios
- **Elenco** - atores, diretores e informações de produção
- **Avaliações** - notas do IMDb, Rotten Tomatoes, etc.

O Jellyfin busca esses dados automaticamente no [TMDb](https://www.themoviedb.org/) e no [OMDb](https://www.omdbapi.com/). Para isso funcionar, ele precisa identificar corretamente o filme ou série e é aí que a nomenclatura dos arquivos entra.

### Filmes

O formato recomendado é:

```
Nome do Filme (Ano).extensão
```

Exemplos:

```
✅ Constantine (2005).mkv
✅ Back to the Future (1985).mkv
❌ constantine.mkv
❌ Back_to_Future.mkv
```

Cada filme em sua própria pasta, nome com ano entre parênteses, sem underscores.

### Séries e animes

```
Nome da Série (Ano)/Season XX/SXXEXX - Título.extensão
```

Exemplos:

```
✅ Breaking Bad (2008)/Season 01/S01E01.mkv
✅ Berserk (1997)/Season 01/S01E01.mkv
❌ breaking.bad.s01e01.mkv
```

Regras:
- Pasta principal com nome e ano
- Subpastas `Season 01`, `Season 02`, etc.
- Episódios no formato `S01E01`
- Título do episódio é opcional
- O formato `1x01` também funciona, mas `S01E01` é mais confiável

> O Jellyfin é flexível com nomenclatura, mas seguir o padrão evita problemas com metadados. Teste primeiro, renomeie só se necessário.

---

## Parte 5 - Criar as bibliotecas

Com os arquivos organizados, vamos configurar as bibliotecas no Jellyfin.

Acesse **Admin Dashboard → Libraries → Add Media Library**.

### Biblioteca de filmes

| Configuração | Definição |
| :--- | :--- |
| Content type | Movies |
| Display name | Filmes |
| Folders | `/home/berserk/media/filmes` |
| Preferred language | Portuguese (Brazil) |

Em **Library Options**:

- ✅ Enable real-time monitoring (detecta novos arquivos automaticamente)
- ❌ Prefer embedded titles over filenames

**Metadata Downloaders** - mantenha a ordem padrão:
1. The Movie Database (TMDb)
2. Open Movie Database (OMDb)

**Image Fetchers** - mantenha padrão (TMDb, OMDb, Screen Grabber).

**Save artwork into media folders** - ✅ marque. Isso salva pôsteres junto dos arquivos, facilitando backups.

Clique em **OK**. O scan inicia automaticamente.

### Biblioteca de séries

Mesmo processo, mudando:

| Configuração | Definição |
| :--- | :--- |
| Content type | Shows |
| Display name | Séries |
| Folders | `/home/berserk/media/series` |

### Biblioteca de animes

| Configuração | Definição |
| :--- | :--- |
| Content type | Shows |
| Display name | Animes |
| Folders | `/home/berserk/media/animes` |

Repita para quantas bibliotecas precisar. Separar por tipo facilita o controle de acesso por usuário depois.

Após criar todas, clique em **Scan All Libraries** e aguarde o processamento.

---

## Parte 6 - Configurar transcodificação de hardware

Aqui está o que diferencia um setup funcional de um setup realmente bom. Sem aceleração de hardware, cada stream transcodificado massacra a CPU. Com Quick Sync, o N5105 lida com isso tranquilamente.

### 6.1 Verificar compatibilidade

Essa configuração é para processadores Intel com GPU integrada (iGPU). A maioria dos chips Intel modernos tem:

- Celeron, Pentium, Core i3/i5/i7/i9 com iGPU
- **Gen 9 ou superior** (Skylake, Kaby Lake, Coffee Lake, Ice Lake, Tiger Lake, Alder Lake, Jasper Lake...)

O N5105 é Jasper Lake (Gen 11), então está bem coberto. Se você tiver AMD ou NVIDIA, o processo de passthrough é parecido, mas os drivers e as opções no Jellyfin são diferentes. 

> **Deixei o link para a documentação oficial do Jellyfin no final do post.**

### 6.2 Habilitar GuC/HuC no host

GuC (Graphics micro-Controller) e HuC (HuC micro-Controller) são micro-controladores embarcados na GPU Intel. Por padrão ficam desabilitados no kernel. Habilitá-los melhora a eficiência de transcodificação - o HuC em particular cuida da decodificação de vídeo por hardware.

No host Proxmox:

```bash
nano /etc/modprobe.d/i915.conf
```

Adicione:

```
options i915 enable_guc=3
```

O valor `3` habilita tanto o GuC quanto o HuC simultaneamente. Com `1` só o GuC; com `2` só o HuC. Para transcodificação de vídeo, o `3` é o correto.

Reinicie o host:

```bash
reboot
```

### 6.3 Passar a GPU para o container

Edite a configuração do container no host:

```bash
nano /etc/pve/lxc/102.conf
```

Adicione ao final do arquivo:

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

Isso permite que o container acesse os dispositivos da GPU (`card0` para display, `renderD128` para renderização).

> Se o host tiver múltiplas GPUs, os números podem ser diferentes. Use `ls -la /dev/dri/` no host para verificar.

Reinicie o container:

```bash
pct stop 102
pct start 102
```

### 6.3 Configurar permissões e drivers

Dentro do container:

```bash
# Verificar se os dispositivos aparecem
ls -la /dev/dri/

# Adicionar o usuário jellyfin aos grupos necessários
usermod -aG video jellyfin
usermod -aG render jellyfin
usermod -aG input jellyfin

# Instalar drivers VA-API
apt install -y vainfo intel-media-va-driver-non-free

# Testar - deve listar perfis de codificação/decodificação
vainfo

# Reiniciar o Jellyfin
systemctl restart jellyfin
```

O `vainfo` deve retornar uma lista de profiles (H264, HEVC, VP9, etc.). Se retornar erro, verifique se os dispositivos `/dev/dri/` estão acessíveis.

### 6.4 Ativar no Jellyfin

Acesse **Admin Dashboard → Playback → Transcoding**:

| Configuração | Definição |
| :--- | :--- |
| Hardware acceleration | Intel Quick Sync (QSV) |
| Enable hardware decoding for | H264, HEVC, MPEG2, VC1, VP8, VP9 |
| Enable hardware encoding | ✅ |
| Enable Intel Low-Power H.264 hardware encoder | ✅ |
| Enable Intel Low-Power HEVC hardware encoder | ✅ |
| Prefer OS native VA-API hardware decoders | ✅ |

> **AV1:** não marque. O N5105 (Jasper Lake) não suporta decodificação AV1 por hardware.
Só marque se tiver GPU mais recente/compatível.

**Transcoding thread count**: 2-4 (não use mais que o número de núcleos físicos).

Clique em **Save**.

---

## Parte 7 - Testar e verificar

### Reproduzir conteúdo

Acesse o Jellyfin de outro dispositivo e reproduza um vídeo. Se o dispositivo não suportar o formato original, a transcodificação acontece automaticamente.

### Verificar nos logs

Em **Admin Dashboard → Logs → FFmpeg**, procure por:

```
Stream #0:0 -> #0:0 (mpeg4 -> h264_qsv)
```

O `h264_qsv` confirma que o Quick Sync está sendo usado. Se aparecer `h264 (native)` ou similar, a transcodificação está em software.

### Monitorar uso de CPU

No Proxmox, selecione o container Jellyfin e observe o gráfico de CPU:

- **Sem aceleração**: 80-90% de CPU
- **Com Quick Sync**: 10-15% de CPU

A diferença é brutal.

---

## Troubleshooting

**GPU não aparece no container:**

```bash
# No host - verificar se o driver está carregado
lsmod | grep i915

# Se não aparecer, verificar se o módulo i915 está configurado
cat /etc/modprobe.d/i915.conf
```

**vainfo retorna erro:**

```bash
# Reinstalar drivers
apt install -y intel-media-va-driver-non-free libva2 vainfo
```

**Jellyfin sem acesso à GPU:**

```bash
# Verificar grupos do usuário jellyfin
groups jellyfin
# Deve mostrar: jellyfin video render input

# Reiniciar
systemctl restart jellyfin
```

**Container não inicia após configuração da GPU:**

```bash
# No host - comentar as linhas lxc.* adicionadas
nano /etc/pve/lxc/102.conf
# Adicionar # no início das linhas problemáticas
pct start 102
```

**Transcodificação ainda usa CPU:**
1. Verifique se as configurações de Playback estão salvas
2. Confirme que o dispositivo cliente não está em Direct Play (nesse caso não transcodifica - o que é bom)
3. Verifique os logs do FFmpeg

---

## Resumo da configuração

```
Host Proxmox (N5105):
├── /zpool/jellyfin/media          ← dataset ZFS nos WD Red Plus 4TB (mirror)
├── /etc/pve/lxc/102.conf          ← config do container com passthrough GPU
└── /etc/modprobe.d/i915.conf      ← enable_guc=3

Container LXC (ID 102 - jellyfin):
├── /home/berserk/media/           ← mount point do dataset ZFS
│   ├── filmes/
│   ├── series/
│   ├── animes/
│   └── shows/
├── /dev/dri/card0                 ← GPU Intel passthrough
├── /dev/dri/renderD128            ← renderização
└── Jellyfin → http://IP:8096
```

### Comandos-chave

```bash
# Criar dataset
zfs create zpool/jellyfin/media

# Montar no container
pct set 102 --mp2 /zpool/jellyfin/media,mp=/home/berserk/media

# Passar GPU (editar /etc/pve/lxc/102.conf)

# Configurar usuário
usermod -aG video,render,input jellyfin

# Testar GPU
vainfo
```

---

## Manutenção

### Atualizar o Jellyfin

```bash
apt update
apt upgrade jellyfin
systemctl restart jellyfin
```

### Backup da configuração

A configuração do Jellyfin fica em dois diretórios:

```bash
# Backup manual
tar -czf jellyfin-backup.tar.gz /etc/jellyfin /var/lib/jellyfin
```

Ou use o backup do próprio Proxmox para fazer snapshot do container inteiro.

### Monitorar logs

```bash
# Logs em tempo real
journalctl -u jellyfin -f

# Ou pela interface web: Admin Dashboard → Logs
```

---

## Recursos adicionais

- [Documentação Oficial Jellyfin](https://jellyfin.org/docs/)
- [Hardware Acceleration - Jellyfin](https://jellyfin.org/docs/general/administration/hardware-acceleration/)
- [Proxmox LXC - Privileged Containers](https://pve.proxmox.com/wiki/Linux_Container)
- [Intel Quick Sync Compatibility](https://en.wikipedia.org/wiki/Intel_Quick_Sync_Video)
- [AMD VA-API Support](https://wiki.archlinux.org/title/Hardware_video_acceleration)

---

Com o Jellyfin rodando e a transcodificação de hardware funcionando, o home server já cumpre seu primeiro objetivo: um servidor de mídia próprio, sem depender de streaming, sem catálogo sumindo, sem assinatura.

O próximo passo é configurar aplicativos clientes, criar usuários com permissões separadas e adicionar plugins úteis como OpenSubtitles para legendas automáticas.
