---
title: "DNS Spoofing via MITM baseado em ARP"
date: 2026-02-02
layout: single
lang: pt-BR
translation_key: dns-spoofing
categories: [Laboratórios]
tags: [DNS, Spoofing, ARP, MITM, Bettercap, Wireshark]
excerpt: "Demonstração de falsificação de respostas DNS a partir de uma posição man-in-the-middle criada com ARP spoofing em um laboratório Proxmox controlado."
permalink: /pt/laboratorios/dns-spoofing/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Uma pequena falha na confiança pode permitir a falsificação de respostas DNS e o redirecionamento de tráfego."
  image_height: 300px
author_profile: true
author: julia_pt
published: false
sidebar:
  nav: "sidebar_pt"
---

<em>
<strong>Importante:</strong> esta técnica é apresentada somente para fins educacionais. O teste foi realizado em um laboratório privado e controlado. Executar ARP spoofing ou DNS spoofing em uma rede sem autorização explícita é ilegal e antiético.
</em>

### Visão geral

---

Este projeto demonstra a **falsificação de respostas DNS a partir de uma posição man-in-the-middle baseada em ARP spoofing**.

Primeiro, a máquina Kali utiliza ARP spoofing para se posicionar entre uma vítima Windows e o servidor DNS local. Em seguida, a máquina atacante intercepta consultas DNS, envia respostas falsificadas que associam um hostname legítimo a um endereço controlado pelo atacante e bloqueia as respostas enviadas pelo resolvedor legítimo.

O objetivo do laboratório é demonstrar:

1. o funcionamento normal da resolução DNS antes do ataque;
2. a criação de uma posição MITM por meio de ARP spoofing;
3. a injeção de respostas DNS falsificadas;
4. os artefatos de rede e de endpoint disponíveis para defensores;
5. técnicas de detecção, mitigação e limpeza.

Todos os testes foram realizados em um laboratório **Proxmox**, com os sistemas conectados à mesma rede virtual isolada.

> Este laboratório dá continuidade ao projeto de ARP spoofing, que explica com mais detalhes como a máquina atacante obteve a posição man-in-the-middle utilizada aqui.
>
> [Ler o laboratório de ARP Spoofing](/pt/laboratorios/arp-spoofing/){: target="\_blank" rel="noopener noreferrer" }

### Estrutura do laboratório

---

| Função               | Hostname          | Descrição                                                      |
| -------------------- | ----------------- | -------------------------------------------------------------- |
| **WINDOWS10_VICTIM** | Alvo de teste     | Cliente Windows 10 que realiza consultas DNS                   |
| **KALI_ATTACKER**    | Máquina atacante  | Executa ARP spoofing, Bettercap, iptables e captura de pacotes |
| **DNS_SERVER**       | Raspberry Pi      | Resolvedor DNS local usando `dnsmasq`                          |
| **UBUNTU_SERVER**    | Servidor legítimo | Hospeda o serviço legítimo em `portal.company.local`           |

**Identificadores utilizados**

Os seguintes nomes substituem os endereços reais utilizados no laboratório:

- `WINDOWS10_VICTIM_IP`
- `KALI_ATTACKER_IP`
- `DNS_SERVER_IP`
- `UBUNTU_SERVER_IP`

### O que é DNS spoofing?

---

DNS spoofing é um ataque em que um adversário envia uma resposta DNS falsificada para fazer com que um domínio resolva para um endereço IP incorreto ou controlado pelo atacante.

Neste laboratório, o hostname legítimo:

```text
portal.company.local
```

deveria resolver para o servidor Ubuntu. Durante o ataque, a máquina Kali envia uma resposta falsificada apontando esse hostname para o endereço da própria máquina atacante.

DNS spoofing também pode ser chamado, de forma mais ampla, de **DNS poisoning**. Entretanto, o termo mais preciso para este laboratório é **falsificação de resposta DNS**, porque o atacante injeta uma resposta falsa durante a comunicação. O cache do resolvedor DNS Raspberry Pi não é diretamente modificado.

A cadeia do ataque é:

```text
ARP spoofing
      ↓
Kali obtém uma posição MITM
      ↓
A vítima envia uma consulta DNS
      ↓
Kali injeta uma resposta DNS falsificada
      ↓
A resposta legítima é bloqueada
      ↓
A vítima resolve o hostname para o endereço do atacante
```

**Possíveis objetivos do atacante**

Um redirecionamento bem-sucedido poderia ser utilizado para:

- exibir uma página falsa ou maliciosa;
- capturar credenciais inseridas em uma página de login imitada;
- redirecionar usuários para uma infraestrutura que distribui malware;
- injetar ou modificar conteúdo não criptografado;
- monitorar ou interromper comunicações;
- apoiar outras atividades man-in-the-middle.

Criptografia e validação de certificados reduzem significativamente a utilidade desse ataque. Por exemplo, em uma conexão HTTPS, o atacante precisaria apresentar um certificado válido para o hostname legítimo. Caso contrário, o navegador deveria exibir um alerta de certificado.

### Pré-requisitos

---

Antes do teste:

1. configure o Raspberry Pi ou uma máquina Linux como resolvedor DNS local;
2. configure a vítima Windows para utilizar apenas `DNS_SERVER_IP` como servidor DNS;
3. hospede o serviço legítimo em `UBUNTU_SERVER`;
4. adicione o hostname ao arquivo de configuração do `dnsmasq`;
5. confirme que todos os sistemas estão conectados à rede isolada do laboratório Proxmox.

Na vítima Windows, verifique a configuração DNS:

```powershell
ipconfig /all
```

O campo **DNS Servers** deve mostrar somente o endereço do Raspberry Pi.

### Baseline A — resolução DNS normal

---

Antes de executar o ataque, documentei o fluxo normal de comunicação.

O hostname `portal.company.local` foi configurado no `dnsmasq` do Raspberry Pi para resolver para `UBUNTU_SERVER_IP`.

![Baseline A — configuração do dnsmasq](/assets/images/dns-poisoning/RASPBERRY_PI_DNSMASQ.CONF_BASE_A.png)

Na máquina `WINDOWS10_VICTIM`, executei:

```powershell
nslookup portal.company.local
```

A resposta retornou o endereço legítimo do servidor Ubuntu.

![Baseline A — resposta legítima do nslookup](/assets/images/dns-poisoning/BASE_A_NSLOOKUP_W10_TO_UBUNTU_BASE_A.png)

O comportamento esperado foi estabelecido como:

```text
portal.company.local → UBUNTU_SERVER_IP
```

Esse baseline é importante porque fornece um resultado conhecido e confiável para comparação com a resposta falsificada observada durante o ataque.

### Baseline B — DNS spoofing via MITM baseado em ARP

---

**Etapa 1 — estabelecer a posição man-in-the-middle**

Primeiro, a máquina Kali precisa se posicionar entre a vítima Windows e o servidor DNS.

Os seguintes comandos envenenam o cache ARP dos dois endpoints:

```bash
sudo arpspoof -i eth0 -t WINDOWS10_VICTIM_IP DNS_SERVER_IP
```

```bash
sudo arpspoof -i eth0 -t DNS_SERVER_IP WINDOWS10_VICTIM_IP
```

O primeiro comando informa à vítima Windows que o endereço IP do servidor DNS está associado ao endereço MAC da máquina Kali.

O segundo comando informa ao servidor DNS que o endereço IP da vítima está associado ao MAC da máquina Kali.

O fluxo passa a ser:

```text
WINDOWS10_VICTIM
        ↓
KALI_ATTACKER
        ↓
DNS_SERVER
```

em vez de:

```text
WINDOWS10_VICTIM
        ↓
DNS_SERVER
```

Na máquina Windows, a tabela ARP pode ser verificada com:

```powershell
arp -a
```

O endereço IP do servidor DNS passou a aparecer associado ao endereço MAC da máquina atacante, confirmando o envenenamento do cache ARP da vítima.

![Baseline B — tabela ARP envenenada no Windows](/assets/images/dns-poisoning/ARP_TABLE_WINDOWS10_ARP_SPOOF_BASE_B.png)

Essa alteração na tabela ARP estabelece a posição de interceptação. Sozinha, ela ainda não altera a resolução DNS.

**Etapa 2 — configurar o DNS spoofing no Bettercap**

Em seguida, configurei o Bettercap para responder a consultas destinadas a:

```text
portal.company.local
```

com o endereço IP da máquina atacante.

```bash
sudo bettercap -iface eth0 -eval "
set dns.spoof.domains portal.company.local;
set dns.spoof.address KALI_ATTACKER_IP;
set dns.spoof.all true;
dns.spoof on;
events.stream on;"
```

Como o ARP spoofing já estava sendo executado pelo `arpspoof`, utilizei o Bettercap principalmente para gerar a resposta DNS falsificada.

O Bettercap também possui funcionalidades próprias de ARP spoofing:

```text
set arp.spoof.targets WINDOWS10_VICTIM_IP,DNS_SERVER_IP
set arp.spoof.fullduplex true
arp.spoof on
```

No entanto, essa configuração não funcionou de forma confiável no meu laboratório. Por isso, executei o ARP spoofing separadamente com `arpspoof`.

A partir desse momento, a máquina Kali podia observar a consulta DNS da vítima e injetar uma resposta que associava o hostname legítimo a `KALI_ATTACKER_IP`.

**Etapa 3 — bloquear a resposta DNS legítima**

O servidor DNS legítimo ainda poderia enviar sua própria resposta. Isso poderia produzir respostas concorrentes e tornar o resultado inconsistente.

Para garantir que apenas a resposta falsificada chegasse à vítima, adicionei regras temporárias de encaminhamento na máquina Kali:

```bash
sudo iptables -I FORWARD \
  -p udp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

```bash
sudo iptables -I FORWARD \
  -p tcp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

Essas regras bloqueiam as respostas DNS enviadas pelo resolvedor legítimo para a vítima por UDP e TCP na porta 53.

O fluxo resultante é:

```text
A vítima envia uma consulta DNS
      ↓
Kali observa a consulta
      ↓
Kali envia uma resposta falsificada
      ↓
A resposta do resolvedor legítimo é bloqueada
      ↓
A vítima aceita a resposta controlada pelo atacante
```

### Resultado do ataque

---

Na máquina `WINDOWS10_VICTIM`, executei novamente:

```powershell
nslookup portal.company.local
```

O hostname deixou de resolver para o endereço legítimo do servidor Ubuntu e passou a apontar para o endereço controlado pela máquina atacante.

![Baseline B — resposta falsificada no nslookup](/assets/images/dns-poisoning/NSLOOKUP_W10_BASE_B.png)

A comparação principal é:

```text
Antes do ataque:
portal.company.local → UBUNTU_SERVER_IP

Durante o ataque:
portal.company.local → KALI_ATTACKER_IP
```

Essa alteração confirma que a vítima aceitou a resposta DNS falsificada.

O formato IPv6 mapeado observado na captura também representou uma anomalia que poderia ser considerada durante a investigação. No entanto, esse formato isoladamente não comprova DNS spoofing e deve ser correlacionado com outras evidências.

### Evidências em captura de pacotes

---

**Resposta DNS falsificada**

A captura do Wireshark na máquina Kali mostra o tráfego DNS relacionado à consulta da vítima por `portal.company.local`.

![Wireshark — resposta DNS falsificada](/assets/images/dns-poisoning/WIRESHARK_KALI_DNS_BASEB.png)

Durante a análise, é importante verificar:

- o hostname consultado;
- o ID da transação DNS;
- a origem da resposta;
- o endereço IP retornado;
- a existência de múltiplas respostas para a mesma consulta;
- o intervalo entre a resposta falsificada e a resposta legítima.

Um filtro útil no Wireshark é:

```text
dns.qry.name == "portal.company.local"
```

Um defensor poderia comparar o endereço retornado com:

- o registro DNS interno esperado;
- respostas fornecidas por outro resolvedor confiável;
- capturas anteriores conhecidas como legítimas;
- logs do servidor DNS;
- entradas do cache DNS do endpoint.

**ICMP Redirects observados durante o teste**

Também foram observadas mensagens ICMP Redirect durante a captura:

![Wireshark — ICMP Redirects](/assets/images/dns-poisoning/WIRESHARK_KALI_BASE_B.png)

Mensagens ICMP Redirect normalmente são enviadas por roteadores para informar a um host que existe uma rota mais apropriada.

Quando essas mensagens são originadas por um endpoint inesperado ou por um dispositivo que não deveria atuar como roteador, elas podem indicar:

- comportamento anormal de roteamento;
- configuração incorreta;
- encaminhamento inesperado de tráfego;
- tentativa de manipulação do caminho de rede.

Neste laboratório, o tráfego ICMP Redirect deve ser tratado como uma anomalia complementar, e não como prova isolada de DNS spoofing.

A investigação deve correlacioná-lo com evidências mais fortes, como:

- alterações inesperadas na tabela ARP;
- associações duplicadas entre IP e MAC;
- respostas DNS falsificadas;
- respostas diferentes das fornecidas pelo resolvedor confiável;
- tráfego passando por um endpoint não autorizado;
- mensagens ICMP Type 5 originadas por um host inesperado.

### Oportunidades de detecção

---

**Indicadores na camada ARP**

- O endereço IP do servidor DNS passa a estar associado a outro endereço MAC.
- O mesmo MAC aparece associado a vários endereços IP.
- As associações ARP mudam repetidamente em um curto período.
- Um endpoint comum envia uma quantidade incomum de respostas ARP.
- A vítima e o servidor DNS associam o endereço um do outro ao MAC da máquina atacante.

**Indicadores na camada DNS**

- Um hostname resolve para um endereço IP inesperado.
- Diferentes resolvedores retornam respostas conflitantes.
- Existem múltiplas respostas DNS para a mesma consulta.
- A resposta DNS é originada por uma fonte inesperada.
- O TTL ou a estrutura da resposta difere do baseline conhecido.
- Um serviço interno passa a resolver para um workstation ou host desconhecido.

**Indicadores de rede**

- O tráfego DNS passa por um endpoint que não é roteador ou dispositivo de segurança autorizado.
- Um host inesperado gera mensagens ICMP Redirect.
- As respostas do resolvedor legítimo deixam de chegar à vítima.
- Surgem regras de encaminhamento ou firewall inesperadas em um endpoint.
- Há aumento simultâneo de anomalias ARP, DNS e ICMP no mesmo segmento.

**Indicadores no endpoint**

- O navegador exibe alertas de certificado ao acessar um hostname conhecido.
- Usuários são redirecionados para páginas desconhecidas.
- O cache DNS local contém endereços inesperados.
- Ferramentas de segurança detectam mudanças nas associações ARP.
- Aplicações se conectam a um endereço IP diferente do esperado.

### Mitigação

---

A proteção mais efetiva utiliza controles em diferentes camadas.

**DHCP snooping**

O DHCP snooping cria uma tabela confiável contendo:

- endereço IP;
- endereço MAC;
- VLAN;
- porta do switch;
- concessão DHCP.

Essa tabela pode ser utilizada pelo Dynamic ARP Inspection para determinar se uma mensagem ARP é legítima.

**Dynamic ARP Inspection**

O Dynamic ARP Inspection valida pacotes ARP comparando-os com associações confiáveis entre IP, MAC, VLAN e porta.

Mensagens ARP falsificadas podem ser descartadas antes de chegarem aos endpoints.

O DAI é uma das proteções mais diretas contra a etapa de ARP spoofing desse ataque.

**Port security**

Port security limita quais endereços MAC podem utilizar cada porta de um switch gerenciável.

Isso reduz a possibilidade de dispositivos não autorizados se conectarem à rede, embora não impeça necessariamente ataques originados por um endpoint que já esteja autorizado.

**VLANs e regras de firewall**

VLANs separam dispositivos em diferentes domínios de broadcast de camada 2, limitando o alcance do ARP spoofing.

Regras de firewall e ACLs devem controlar a comunicação entre as VLANs.

VLANs sozinhas não impedem ataques entre sistemas localizados dentro da mesma VLAN.

**DNSSEC**

O DNSSEC permite que resolvedores validem assinaturas criptográficas associadas aos registros DNS.

Uma resposta falsificada que não apresente uma assinatura válida pode ser rejeitada por um resolvedor com validação habilitada.

A proteção depende de:

- a zona DNS estar assinada;
- o resolvedor realizar a validação;
- a cadeia de confiança estar configurada corretamente.

**DNS-over-HTTPS e DNS-over-TLS**

DoH e DoT criptografam o tráfego DNS entre o cliente e um resolvedor confiável.

Isso dificulta que um atacante presente na rede local leia ou modifique consultas e respostas DNS durante o trânsito.

Esses protocolos devem ser configurados para utilizar um resolvedor aprovado e confiável.

**HTTPS e HSTS**

HTTPS não impede a falsificação DNS, mas reduz a utilidade do redirecionamento.

Mesmo que a vítima seja direcionada para um servidor falso, o atacante ainda precisaria apresentar um certificado TLS válido para o hostname legítimo.

Sem um certificado válido, o navegador deveria exibir um alerta.

HSTS ajuda a impedir que o navegador aceite silenciosamente o rebaixamento de HTTPS para HTTP.

**Monitoramento de ARP e DNS**

Ferramentas de monitoramento podem identificar:

- alterações inesperadas entre IP e MAC;
- respostas DNS conflitantes;
- respostas DNS duplicadas;
- respostas originadas por fontes não autorizadas;
- atividade incomum de ICMP Redirect.

Possíveis ferramentas e fontes de dados incluem:

- Arpwatch;
- Suricata;
- Snort;
- Zeek;
- Wireshark;
- logs de switches;
- logs do resolvedor DNS;
- logs de firewall;
- regras de correlação no SIEM.

### Defesa em profundidade

---

As mitigações funcionam melhor quando combinadas:

```text
DHCP snooping estabelece associações confiáveis entre IP e MAC
      ↓
Dynamic ARP Inspection bloqueia mensagens ARP falsificadas
      ↓
Port security limita dispositivos não autorizados
      ↓
VLANs e firewall restringem o alcance do ataque
      ↓
DNSSEC ou DNS criptografado protegem a resolução de nomes
      ↓
HTTPS e SSH protegem os dados das aplicações
      ↓
Monitoramento identifica comportamentos suspeitos
```

Nenhum controle elimina completamente o risco sozinho.

As proteções ARP tentam impedir que o atacante obtenha a posição MITM. A segmentação reduz a área afetada. Os controles DNS protegem a resolução de nomes. A criptografia protege o conteúdo mesmo quando ocorre interceptação. O monitoramento fornece visibilidade quando os controles preventivos falham.

### Resposta imediata

---

Quando houver suspeita de ARP spoofing ou DNS spoofing:

1. isolar o dispositivo suspeito;
2. registrar as tabelas ARP atuais antes de limpá-las;
3. capturar o tráfego ARP e DNS relevante;
4. comparar as respostas DNS com um resolvedor confiável;
5. revisar logs de DHCP, switch, firewall e servidor DNS;
6. limpar os caches ARP e DNS afetados;
7. verificar se permaneceram configurações de rede não autorizadas;
8. redefinir credenciais caso usuários as tenham inserido em uma página redirecionada;
9. investigar alertas de certificado e destinos inesperados;
10. continuar monitorando alterações ARP e DNS recorrentes.

### Limpeza do laboratório

---

Após o teste autorizado, removi as alterações temporárias.

**Na máquina Kali**

Remover a regra de bloqueio DNS sobre UDP:

```bash
sudo iptables -D FORWARD \
  -p udp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

Remover a regra de bloqueio DNS sobre TCP:

```bash
sudo iptables -D FORWARD \
  -p tcp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

Encerrar o Bettercap e os processos de ARP spoofing:

```bash
sudo pkill bettercap
sudo pkill arpspoof
```

**Na máquina Windows**

Limpar o cache DNS:

```powershell
ipconfig /flushdns
```

Restaurar a configuração DNS fornecida por DHCP, quando apropriado:

```powershell
netsh interface ip set dns "Ethernet" dhcp
```

Verificar novamente a resolução:

```powershell
nslookup portal.company.local
```

**No servidor DNS Raspberry Pi**

Reiniciar o `dnsmasq`:

```bash
sudo systemctl restart dnsmasq
```

Após a limpeza, o resultado esperado deve ser restaurado:

```text
portal.company.local → UBUNTU_SERVER_IP
```

### Principais conclusões

---

Este laboratório demonstrou que:

- ARP spoofing pode criar uma posição man-in-the-middle dentro de uma rede local de camada 2;
- consultas DNS sem criptografia, usando UDP ou TCP na porta 53, podem ser observadas e manipuladas a partir dessa posição;
- uma resposta DNS falsificada pode redirecionar a vítima para um endereço controlado pelo atacante;
- bloquear a resposta legítima torna a resposta falsificada mais consistente;
- tabelas ARP, respostas DNS, capturas de pacotes e anomalias de roteamento fornecem evidências úteis para defensores;
- criptografia e validação de certificados reduzem o impacto mesmo quando o redirecionamento ocorre;
- controles de camada 2, validação DNS, segmentação, criptografia e monitoramento devem ser combinados.

### Conclusão

---

Este projeto demonstrou a **falsificação de respostas DNS por meio de um ataque man-in-the-middle baseado em ARP spoofing** em um ambiente Proxmox controlado.

O ataque não modificou o cache do resolvedor DNS Raspberry Pi. Em vez disso, a máquina Kali interceptou o tráfego DNS da vítima, injetou uma resposta falsificada e impediu que a resposta do resolvedor legítimo chegasse ao endpoint.

Por esse motivo, **DNS spoofing** é o termo mais preciso para a técnica demonstrada.

Do ponto de vista defensivo, o laboratório mostra por que eventos DNS não devem ser investigados isoladamente. Respostas DNS inesperadas podem estar relacionadas a manipulações na camada 2, alterações nas tabelas ARP, encaminhamento suspeito de pacotes ou tráfego passando por um intermediário não autorizado.

A mitigação mais forte utiliza uma abordagem em camadas, combinando:

- DHCP snooping;
- Dynamic ARP Inspection;
- port security;
- segmentação por VLAN;
- regras de firewall;
- DNSSEC ou DNS criptografado;
- HTTPS e outros protocolos com criptografia autenticada;
- monitoramento contínuo de ARP, DNS e tráfego de rede.
