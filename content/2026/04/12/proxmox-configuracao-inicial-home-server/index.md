---
title: "Home Server - Primeiras configurações após instalar o Proxmox"
date: 2026-04-12T15:00:00-03:00
slug: "proxmox-configuracao-inicial-home-server"
tags:
  - homelab
  - home-server
  - proxmox
  - linux
  - tutorial
draft: false
description: "O que configurar logo após instalar o Proxmox VE: storage, SMART, VLAN, backups e ISOs - antes de criar a primeira VM."
---

O Proxmox está instalado, atualizado e acessível pela interface web. Antes de criar qualquer VM ou container, é recomendável fazer algumas configurações básicas. Não é necessário fazer tudo de uma vez, mas cada item aqui evita problemas futuros.

Este post descreve as configurações que fiz no meu home server logo após a instalação. É uma visão geral, sugestão do que fazer.

## Reconhecer os discos e criar o armazenamento

O primeiro passo é garantir que o Proxmox enxergue todos os discos e que eles estejam disponíveis como armazenamento.

Acesse a seção **Node → Disks** e verifique a lista de discos físicos com modelo, tamanho e status SMART. Se algum disco não aparecer, o problema geralmente está na interface da placa-mãe ou no cabo.

Com os discos visíveis, é hora de criar os pools de armazenamento. O Proxmox suporta ZFS nativamente e você pode criar pools diretamente pela interface web em **Disks → ZFS → Create ZFS** ou via terminal.

No meu caso, criei dois pools:

- **local-zfs** → Kingston NV2 1TB (nvme0n1) - instalação do Proxmox, VMs e ISOs
- **zpool** → 2x WD Red Plus 4TB (sda + sdb) - dados, backups e mídia

O pool dos WDs é um mirror ZFS: os dois HDs são espelhados, então a capacidade visível é de ~4TB - não 8TB. 

> A vantagem é a redundância. Se um HD morrer, não perde dados. Vou detalhar a criação do mirror no próximo post.

Após criar os pools, eles aparecem em **Datacenter → Storage**. Verifique se os tipos de conteúdo estão configurados corretamente para cada pool: ISO images, VM disks, VZDump backup file.

## Verificar o SMART dos discos

O SMART monitora a saúde dos discos e é a primeira linha de defesa contra falha de hardware. No Proxmox, fica em **Node → Disks**, a coluna **Health** mostra o status de cada disco.

O que você deseja ver é **PASSED** em todos. Se aparecer **FAILED** ou **Unknown**, investigue antes de seguir.

Para detalhes via SSH:

```bash
# Relatório completo de um disco
smartctl -a /dev/sda

# Teste rápido (~2 minutos)
smartctl -t short /dev/sda

# Resultado do teste
smartctl -l selftest /dev/sda
```

Preste atenção nos campos **Reallocated_Sector_Ct**, **Current_Pending_Sector** e **Offline_Uncorrectable**, que indicam problemas potenciais no disco. Qualquer valor diferente de zero nesses três merece atenção e pode sinalizar a necessidade de substituição do disco. 

> **Discos novos devem estar zerados.**

## Tornar a bridge VLAN-aware

Por padrão, o Proxmox cria uma bridge `vmbr0` que conecta todas as VMs na mesma rede. Isso funciona, mas se você tem múltiplas NICs ou quer segmentar o tráfego por serviço, habilitar VLAN-awareness na bridge vale a pena.

Com VLAN-aware ativado, você pode atribuir uma VLAN tag diferente para cada VM - separar a rede de gerenciamento do tráfego de mídia, por exemplo.

Para ativar: 

- **Node → Network → vmbr0 → Edit → marcar VLAN Aware → OK → Apply Configuration**.

No meu setup o N5105 tem 4 NICs de 2.5G. A `vmbr0` usa a `enp1s0` como uplink para o roteador. As outras três ficam disponíveis para uso futuro - seja uplink dedicado para VMs específicas, seja tráfego de storage em rede isolada.

Se você tem switch gerenciável com suporte a 802.3ad, também é possível agregar duas NICs em um bond LACP para dobrar o bandwidth disponível. A configuração envolve criar um `bond0` com as interfaces desejadas e usá-lo como porta da bridge no lugar da NIC direta.

## Configurar backups automáticos

Antes de criar qualquer VM, configure o agendamento de backup. É muito mais fácil fazer isso agora do que depois de já ter serviços rodando.

Acesse **Datacenter → Backup → Add**:

| Campo | Valor recomendado |
|---|---|
| Node | Seu nó (ex: `pve`) |
| Storage | Onde salvar (ex: pool de dados) |
| Day of week | Dias que fizer sentido |
| Start Time | Horário de baixo uso (ex: 03:00) |
| Compression | ZSTD |
| Mode | Snapshot |

O modo **Snapshot** é o ideal - faz o backup sem parar a VM. O modo **Stop** é mais confiável para consistência de dados, mas interrompe o serviço durante o processo.

Com o storage configurado e o agendamento definido, o Proxmox cuida do resto automaticamente.

## Fazer upload das ISOs

Para criar VMs, você precisa das ISOs dos sistemas operacionais disponíveis no storage. O Proxmox não baixa ISOs automaticamente, você precisa fazer upload ou informar uma URL.

Acesse **Storage → local (ou o pool destinado para ISOs) → ISO Images → Upload** ou **Download from URL**.

ISOs que vale ter antes de começar:

- **Debian 12** - base sólida para serviços Linux, bem testada com Proxmox
- **Ubuntu Server 24.04 LTS** - boa compatibilidade, vasta documentação
- **VirtIO drivers** - obrigatório se for criar VMs Windows. Download direto em [fedorapeople.org/groups/virt/virtio-win/](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/)

Os drivers VirtIO são necessários porque o Windows não reconhece nativamente os adaptadores paravirtualizados do QEMU/KVM. Sem eles, a VM Windows não vê o disco nem a rede durante a instalação.

## IOMMU e PCI Passthrough

IOMMU é a tecnologia que permite passar um dispositivo de hardware físico diretamente para uma VM, como se o dispositivo estivesse plugado diretamente nela, sem intermediário de software.

O caso de uso mais comum é GPU passthrough para VMs Windows, ou passar uma controladora de armazenamento diretamente para uma VM de NAS.

Para habilitar no Proxmox com Intel:

```bash
# Editar parâmetros do GRUB
nano /etc/default/grub

# Alterar a linha GRUB_CMDLINE_LINUX_DEFAULT para:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# Aplicar
update-grub
```

Depois, adicionar os módulos VFIO em `/etc/modules`:

```
vfio
vfio_iommu_type1
vfio_pci
```

Reiniciar e verificar:

```bash
dmesg | grep -e DMAR -e IOMMU
```

Se retornar linhas com `IOMMU enabled`, está ativo.

**Ressalva para CPUs Intel N-series (N5105, N100, N200):** essas CPUs têm suporte a VT-d no papel, mas o comportamento varia conforme a placa-mãe. O N5105 da Topton tem IOMMU groups pouco granulares - dispositivos que deveriam estar em grupos separados ficam agrupados, o que limita o passthrough. Faça os testes, mas não conte com isso como feature principal do setup.

---

Com isso o ambiente está devidamente preparado: discos organizados, saúde monitorada, rede configurada, backups agendados e ISOs disponíveis. O próximo passo é a configuração detalhada do ZFS Mirror com os WD Red Plus, criação do pool, scrub programado e snapshots automáticos.