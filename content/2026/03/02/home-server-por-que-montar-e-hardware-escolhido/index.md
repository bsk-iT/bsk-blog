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

Montei um Home Server. Não foi por luxo nem por hobbyismo: foi porque as alternativas que eu usava estavam me custando tempo demais. A ideia ficou parada na cabeça por meses, sei lá, anos. Até que alguns problemas começaram a me irritar o suficiente pra eu finalmente fazer alguma coisa.

## O problema com o streaming

O primeiro foi o streaming. Não que eu seja contra pagar por um serviço, o problema é a imprevisibilidade. Títulos "somem" do catálogo sem aviso. Animes que eu queria assistir novamente de repente não estão mais disponíveis. Séries antigas que nunca chegaram a nenhuma plataforma. Filmes de nicho que existem num catálogo hoje e "somem" amanhã.

> **Quero ter controle sobre o que assisto, quando assisto e em qual dispositivo. Um servidor de mídia próprio resolve isso.**

O segundo motivo foi armazenamento na nuvem, *sobre isso vou falar em um outro post*, e também para estudo. Trabalho com tecnologia e sempre quis ter um ambiente local para testar coisas, subir serviços, experimentar configurações sem depender de nuvem ou de uma máquina que uso no dia a dia.

Com esses objetivos na cabeça, fui pesquisar.

## As opções que encontrei

A pesquisa foi longa: fóruns, vídeos, Reddit, comunidades de homelab. As opções que apareceram com mais frequência foram TrueNAS, UnRAID, CasaOS e OpenMediaVault. Cada uma com um foco diferente: TrueNAS é muito sólido para armazenamento e ZFS, UnRAID é popular mas pago, CasaOS tem interface bonita e é mais voltado para iniciantes, OMV é open-source e bom para NAS simples.

Apareceram também as soluções prontas de hardware: Synology e ASUSTOR. Chegam com tudo junto, hardware e software pensado para quem quer plugar e usar. Para muita gente é a melhor escolha. Mas não era o que eu queria.

Eu queria escolher as peças. Queria entender o que estava rodando embaixo. Queria poder expandir, trocar, configurar do jeito que faz sentido para o meu uso. Soluções prontas tiram exatamente essa parte do processo que me interessa.

## Por que o Proxmox

Foi então que cheguei no **Proxmox VE**. É um hipervisor tipo 1, ou seja, roda direto no hardware sem sistema operacional por baixo, baseado em Debian Linux. Gratuito, open-source, com interface web completa e usado em ambientes corporativos de verdade.

A proposta é simples: em vez de um servidor físico por serviço, você virtualiza tudo. Cada aplicação roda numa VM ou container isolado (*explico mais abaixo*), compartilhando o mesmo hardware. O Proxmox gerencia isso por uma interface web, sem precisar de teclado e monitor conectados à máquina.

Além do uso prático, é um laboratório para estudar virtualização, redes, containers e storage.

```
Uma VM (Máquina Virtual) é um computador "simulado" por software, tem seu sistema operacional próprio, CPU e memória dedicados, como se fosse uma máquina separada existindo dentro da física.

Já o container é mais leve: não simula o hardware inteiro, só isola a aplicação do resto do sistema. Compartilha o núcleo do sistema operacional do host e ocupa muito menos espaço e memória. A troca é que você perde o isolamento total que uma VM oferece.
```

Decisão tomada. Agora era escolher o hardware.

## O hardware escolhido

### Placa-mãe Topton N5105

A Topton N5105 vem com o processador **Intel Celeron N5105** soldado na placa: quatro núcleos, quatro threads, TDP de **10W**. Esse número foi o que me chamou atenção logo de cara. 

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

### SSD MOVESPEED 1TB (NVMe PCIe 4.0 x4)

Um segundo M.2, dedicado a projetos e testes que precisam de disco próprio. No momento está rodando um **node Bitcoin**, que escreve e lê continuamente uma quantidade considerável de dados. Ter um disco separado para isso isola o workload e não compete com o sistema principal.

### 2x WD Red Plus 4TB em ZFS Mirror

Para os dados, filmes, séries e backups, escolhi dois **WD Red Plus de 4TB** em **ZFS Mirror**.

O WD Red Plus é feito para NAS: opera 24/7, 5400 RPM (menos calor e menos ruído que 7200 RPM) e tem firmware otimizado para operação contínua. A escolha do Mirror veio de uma lógica que encontrei durante a pesquisa:

> **Sim, tinha-se feito vários testes com benchmarks, muito bem feito por sinal.**

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

SODIMM, que é o que a placa exige. Dois pentes de 16GB em dual channel, totalizando **32GB**.

Para quem vai virtualizar, RAM é o recurso mais crítico. Cada VM precisa de memória alocada. Com 32GB tenho folga para rodar vários serviços ao mesmo tempo sem o sistema começar a reclamar.

### Fonte Cooler Master G500 Gold 500W

*Sim, 500W para uma placa com TDP de 10W parece exagero. Mas tem uma razão.*

Uma fonte **80 Plus Gold** opera com eficiência acima de 90% na maior parte da curva de carga. Fontes baratas sem certificação perdem mais energia convertendo AC em DC, o que a comunidade chama de **vampirismo energético**: watts que saem da tomada mas nunca chegam nos componentes. Com o servidor rodando 24/7, esse desperdício aparece na conta de luz mês a mês.

Além disso, uma fonte dimensionada com folga trabalha longe do limite, o que significa menos calor e mais vida útil. E sobra margem para adicionar discos ou qualquer outra expansão futura sem precisar trocar a fonte.

### Gabinete Wisecase CK-16

Aqui entra um toque de reutilização. O gabinete é um **Wisecase CK-16**, meu primeiro gabinete de PC Gamer, comprado há muitos anos. Não fazia sentido deixá-lo parado. Acabou sendo uma escolha melhor do que parecia: 4 baias de 5,25", 4 baias internas de 3,5" e mais 2 expostas - **espaço de sobra para os WD Red Plus e qualquer expansão futura**. Gabinetes modernos, costumam vir com uma ou duas baias de HD no máximo. O CK-16 resolve isso sem precisar comprar nada novo.

---

## Resumo do hardware

| Componente | Modelo | Observação | Preço unit. | Total (c/ frete) |
|---|---|---|---|---|
| Placa-mãe / CPU | Topton N5105 | TDP 10W, 4x NIC 2.5G | R$ 627,16 | R$ 627,16 |
| SSD sistema | Kingston NV2 1TB NVMe | Proxmox, VMs, ISOs | R$ 258,99 | R$ 276,31 |
| SSD projetos | MOVESPEED PCIe 4.0 | Node Bitcoin e testes | R$ 236,15 | R$ 236,15 |
| HDs de dados | 2x WD Red Plus 4TB | ZFS Mirror | R$ 444,99 | R$ 889,98 |
| Memória | 2x Corsair Vengeance 16GB DDR4 | 32GB dual channel | R$ 209,99 | R$ 426,33 |
| Fonte | Cooler Master G500 Gold 500W | 80 Plus Gold | R$ 399,90 | R$ 399,90 |
| Gabinete | Wisecase CK-16 | Reutilizado | — | — |
| Cabo Splitter SATA 6P | — | Acessório de instalação | R$ 49,05 | R$ 49,05 |
| **Total** | | | | **R$ 2.904,88** |

---

Com o hardware definido, o próximo passo foi instalar o Proxmox. Isso fica para o próximo post.
