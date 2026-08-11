---
title: "ARP Spoofing: validação de um cenário MITM em laboratório"
date: 2026-01-19
layout: single
lang: pt-BR
translation_key: arp-spoofing
categories: [Laboratórios]
tags: [ARP, Spoofing, MITM, Wireshark]
excerpt: "Laboratório de ARP spoofing em uma rede isolada, com validação da posição man-in-the-middle por tabelas ARP, tcpdump e Wireshark."
permalink: /pt/laboratorios/arp-spoofing/
header:
  teaser: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  image: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  image_description: "Cabos de rede conectados a switches."
  image_height: 300px
author_profile: true
author: julia_pt
sidebar:
  nav: "sidebar_pt"
published: false
---

<em>
<strong>Importante:</strong> este teste foi executado exclusivamente em uma rede privada e controlada de laboratório.
</em>

### Resumo do projeto

Neste laboratório, testei como a manipulação de associações **IP–MAC** por ARP spoofing pode alterar o caminho do tráfego em uma rede local.

O ambiente foi montado no **Proxmox**, e a posição man-in-the-middle foi validada por meio de:

- tabelas ARP;
- `tcpdump`;
- Wireshark.

> Este laboratório estabelece a posição MITM. Em um projeto separado, essa posição é utilizada para testar DNS spoofing.
>
> [Laboratório de DNS Spoofing via MITM baseado em ARP](/pt/laboratorios/dns-spoofing/){: target="\_blank" rel="noopener noreferrer" }

### Ambiente do laboratório

Os três sistemas envolvidos no teste estavam conectados à mesma bridge virtual e à mesma subnet, permitindo comunicação direta em camada 2.

| Sistema              | Função                         |
| -------------------- | ------------------------------ |
| **WINDOWS10_VICTIM** | Cliente Windows 10             |
| **KALI_ATTACKER**    | Host usado para o ARP spoofing |
| **UBUNTU_SERVER**    | Servidor legítimo              |

Um `DNS_SERVER` também fazia parte do ambiente para os testes de resolução DNS.

Todos os sistemas estavam em uma rede privada e isolada. Os endereços reais foram substituídos por identificadores no post.

### Baseline

Antes do teste, validei que a comunicação entre `WINDOWS10_VICTIM` e `UBUNTU_SERVER` ocorria diretamente e que as associações ARP apontavam para os endereços MAC legítimos.

```text
WINDOWS10_VICTIM ←→ UBUNTU_SERVER
```

Também confirmei que `ubuntu.lab` resolvia para o endereço correto do servidor.

![Baseline — resolução de ubuntu.lab no Windows](/assets/images/arp-spoofing/BASE_A_WINDOWS_NSLOOKUP.png)

### Execução controlada do ARP spoofing

Para alterar o caminho do tráfego, usei `arpspoof` na máquina Kali contra os dois endpoints.

O objetivo era modificar as associações nos dois sentidos:

```text
WINDOWS10_VICTIM
IP do servidor → MAC da Kali

UBUNTU_SERVER
IP da vítima → MAC da Kali
```

Na Kali, executei uma instância do `arpspoof` para cada endpoint:

```bash
sudo arpspoof -i eth0 -t WINDOWS10_VICTIM_IP UBUNTU_SERVER_IP
```

```bash
sudo arpspoof -i eth0 -t UBUNTU_SERVER_IP WINDOWS10_VICTIM_IP
```

O primeiro processo fez a vítima associar o IP do servidor ao MAC da Kali. O segundo fez o servidor associar o IP da vítima ao MAC da Kali.

Com as duas associações alteradas, o caminho passou de:

```text
WINDOWS10_VICTIM ←→ UBUNTU_SERVER
```

para:

```text
WINDOWS10_VICTIM
        ↓
KALI_ATTACKER
        ↓
UBUNTU_SERVER
```

Os endpoints IP permaneceram os mesmos; a alteração ocorreu no encaminhamento em camada 2.

### Validação pelas tabelas ARP

A primeira evidência veio das tabelas ARP dos dois endpoints.

No Ubuntu, o IP da máquina Windows passou a apontar para o endereço MAC da Kali.

![Tabela ARP no Ubuntu após a alteração](/assets/images/arp-spoofing/BASE_B_UBUNTU_ARP_2.png)

No Windows, o IP do servidor Ubuntu passou a estar associado ao MAC de `KALI_ATTACKER`.

![Tabela ARP no Windows após a alteração](/assets/images/arp-spoofing/BASE_B_ARP_WINDOWS.png)

Em conjunto:

```text
Windows: IP do servidor → MAC da Kali
Ubuntu:  IP da vítima   → MAC da Kali
```

As duas associações mostraram que o tráfego em camada 2 estava sendo encaminhado para a Kali, em vez de seguir diretamente entre os dois endpoints.

### Validação da interceptação

Depois da alteração das tabelas ARP, observei o tráfego na interface da Kali com `tcpdump`.

A captura mostrou:

- uma requisição HTTP enviada por `WINDOWS10_VICTIM`;
- a resposta HTTP `200` de `UBUNTU_SERVER`;
- os pacotes atravessando a interface de `KALI_ATTACKER`.

![Tráfego observado na máquina Kali com tcpdump](/assets/images/arp-spoofing/BASE_B_KALI_ATTACKER_TCP_DUMP_ARP_SPOOFING.png)

Os endereços IP ainda identificavam a vítima e o servidor como os endpoints da comunicação. A mudança estava no caminho em camada 2:

```text
IP:  WINDOWS10_VICTIM → UBUNTU_SERVER
MAC: WINDOWS10_VICTIM → KALI_ATTACKER → UBUNTU_SERVER
```

Isso confirmou que a Kali estava participando do caminho do tráfego, e não apenas observando broadcasts na rede.

### Conclusão

O laboratório mostrou como alterações nas associações **IP–MAC** podem modificar o caminho do tráfego em camada 2 sem alterar os endpoints IP da comunicação.

A posição MITM foi validada por meio de duas evidências principais:

- as tabelas ARP dos endpoints passaram a associar os IPs legítimos ao endereço MAC da Kali;
- o tráfego entre vítima e servidor foi observado atravessando a interface da máquina intermediária.

Do ponto de vista defensivo, alterações inesperadas nas associações IP–MAC, vários IPs apontando para o mesmo MAC e tráfego passando por um host inesperado são sinais que podem ser correlacionados com tabelas ARP, capturas de rede e telemetria de switches.

Controles como **DHCP snooping**, **Dynamic ARP Inspection (DAI)** e segmentação ajudam a prevenir ou limitar esse comportamento, enquanto protocolos criptografados como **HTTPS e SSH** reduzem o impacto caso o tráfego seja interceptado.
