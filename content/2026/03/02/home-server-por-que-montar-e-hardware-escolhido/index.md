---
title: "Home Server - Por que montar e o hardware escolhido"
date: 2026-03-02T15:00:00-03:00
slug: home-server-por-que-montar-e-hardware-escolhido
tags:
  - homelab
  - home-server
  - proxmox
  - hardware
draft: false
---

Há um tempo venho adiando montar um Home Server. A ideia sempre esteve na cabeça, mas entre uma coisa e outra, nunca saía do papel. Até que dois problemas começaram a me incomodar o suficiente para eu agir.

## O problema com o streaming

O primeiro foi o streaming. Não que eu seja contra pagar por um serviço, o problema é a imprevisibilidade. Títulos "somem" do catálogo sem aviso. Animes que eu queria assistir novamente de repente não estão mais disponíveis. Séries antigas que nunca chegaram a nenhuma plataforma. Filmes de nicho que existem num catálogo hoje e "somem" amanhã.

> **Quero ter controle sobre o que assisto, quando assisto e em qual dispositivo. Um servidor de mídia próprio resolve isso.**

O segundo motivo foi estudo. Trabalho com tecnologia e sempre quis ter um ambiente local para testar coisas, subir serviços, experimentar configurações sem depender de nuvem ou de uma máquina que uso no dia a dia.

Com esses dois objetivos na cabeça, fui pesquisar.

## As opções que encontrei

A pesquisa foi longa: fóruns, vídeos, Reddit, comunidades de homelab. As opções que apareceram com mais frequência foram TrueNAS, UnRAID, CasaOS e OpenMediaVault. Cada uma com um foco diferente: TrueNAS é muito sólido para armazenamento e ZFS, UnRAID é popular mas pago, CasaOS tem interface bonita e é mais voltado para iniciantes, OMV é open-source e bom para NAS simples.

Apareceram também as soluções prontas de hardware: Synology e ASUSTOR. Chegam com tudo junto, hardware e software pensado para quem quer plugar e usar. Para muita gente é a melhor escolha. Mas não era o que eu queria.

Eu queria escolher as peças. Queria entender o que estava rodando embaixo. Queria poder expandir, trocar, configurar do jeito que faz sentido para o meu uso. Soluções prontas tiram exatamente essa parte do processo que me interessa.

## Por que o Proxmox

Foi então que cheguei no **Proxmox VE**. É um hipervisor tipo 1, ou seja, roda direto no hardware sem sistema operacional por baixo, baseado em Debian Linux. Gratuito, open-source, com interface web completa e usado em ambientes corporativos de verdade.

A proposta é simples: em vez de um servidor físico por serviço, você virtualiza tudo. Cada aplicação roda numa VM ou container isolado, compartilhando o mesmo hardware. O Proxmox gerencia isso por uma interface web, sem precisar de teclado e monitor conectados à máquina.

Além do uso prático, é um laboratório para estudar virtualização, redes, containers e storage.

Decisão tomada. Agora era escolher o hardware.

## O hardware escolhido

### Placa-mãe Topton N5105

A Topton N5105 vem com o **Intel Celeron N5105** soldado na placa: quatro núcleos, quatro threads, TDP de **10W**. Esse número foi o que me chamou atenção logo de cara. 

> **Um servidor que fica ligado 24/7 tem que ser eficiente em energia.**

Além do consumo baixo, o que convenceu foi o conjunto de I/O:

- 4 portas de rede 2.5G com Intel i226-V B3, que é essencial para separar tráfego de gerenciamento, VMs e storage em interfaces distintas
- 6x SATA III e 2x M.2, espaço de sobra para crescer em armazenamento
- 2 slots SODIMM DDR4 com suporte até 32GB
- Wake-on-LAN e PXE nativos

É placa de mini PC industrial, feita para rodar sem parar. Sem cooler barulhento, sem peça que não aguenta uso contínuo.

### SSD Kingston NV2 1TB (M.2 NVMe)

O disco do sistema: onde o Proxmox fica, junto com ISOs, containers e VMs. O Kingston NV2 faz leitura de **3500 MB/s** e gravação de **2100 MB/s** via PCIe. Para um homelab, é mais do que o suficiente.

1TB dá espaço confortável antes de precisar repensar a organização do armazenamento.

### SSD MOVESPEED NVMe PCIe 4.0 x4

Um segundo M.2, dedicado a projetos e testes que precisam de disco próprio. No momento está rodando um **node Bitcoin**, que escreve e lê continuamente uma quantidade considerável de dados. Ter um disco separado para isso isola o workload e não compete com o sistema principal.

### 2x WD Red Plus 4TB em ZFS Mirror

Para os dados, filmes, séries e backups, escolhi dois **WD Red Plus de 4TB** em **ZFS Mirror**.

O WD Red Plus é feito para NAS: opera 24/7, gira a 5400 RPM (menos calor e menos ruído que 7200 RPM) e tem firmware otimizado para operação contínua. A escolha do Mirror veio de uma lógica que encontrei durante a pesquisa:

```
Você se importa com seus dados?
Não: vá de striped.
Sim, continue:

Quantos discos você tem?
1:      ZFS não é para você.
2:      Mirror
3-5:    RAIDZ1
6-10:   RAIDZ1 x 2
10-15:  RAIDZ1 x 3
16-20:  RAIDZ1 x 4
```

Com dois discos, Mirror é o caminho. Os 4TB ficam disponíveis para uso (o segundo disco trabalha inteiro como cópia) e se um falhar, não perco nada. O ZFS ainda faz checksum de todos os blocos, então corrupção silenciosa de dados é detectada antes de virar problema.

### 2x Corsair Vengeance 16GB DDR4 2666MHz, 32GB no total

SODIMM, que é o que a placa exige. Duas pentes de 16GB em dual channel, totalizando **32GB**.

Para quem vai virtualizar, RAM é o recurso mais crítico. Cada VM precisa de memória alocada. Com 32GB tenho folga para rodar vários serviços ao mesmo tempo sem o sistema começar a reclamar.

### Fonte Cooler Master G500 Gold 500W

*Sim, 500W para uma placa com TDP de 10W parece exagero. Mas tem uma razão.*

Uma fonte **80 Plus Gold** opera com eficiência acima de 90% na maior parte da curva de carga. Fontes baratas sem certificação perdem mais energia convertendo AC em DC, o que a comunidade chama de **vampirismo energético**: watts que saem da tomada mas nunca chegam nos componentes. Com o servidor rodando 24/7, esse desperdício aparece na conta de luz mês a mês.

Além disso, uma fonte dimensionada com folga trabalha longe do limite, o que significa menos calor e mais vida útil. E sobra margem para adicionar discos ou qualquer outra expansão futura sem precisar trocar a fonte.

---

## Resumo do hardware

| Componente | Modelo | Observação |
|---|---|---|
| Placa-mãe / CPU | Topton N5105 | TDP 10W, 4x NIC 2.5G |
| SSD sistema | Kingston NV2 1TB NVMe | Proxmox, VMs, ISOs |
| SSD projetos | MOVESPEED PCIe 4.0 | Node Bitcoin e testes |
| HDs de dados | 2x WD Red Plus 4TB | ZFS Mirror |
| Memória | 2x Corsair Vengeance 16GB DDR4 | 32GB dual channel |
| Fonte | Cooler Master G500 Gold 500W | 80 Plus Gold |

---

Com o hardware definido, o próximo passo foi instalar o Proxmox. Isso fica para o próximo post.
