---
title: "ARP Spoofing: demonstração de ataque MITM e estratégias de mitigação"
date: 2026-01-19
layout: single
lang: pt-BR
translation_key: arp-spoofing
categories: [Laboratórios]
tags: [ARP, Spoofing, MITM, Wireshark]
excerpt: "Demonstração de ARP spoofing em um cenário man-in-the-middle: comportamento normal, ataque e medidas de mitigação."
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
---

<em>
**Importante — esta técnica é apresentada exclusivamente para fins educacionais e foi executada em um ambiente de laboratório privado. Realizar esse tipo de atividade em uma rede sem autorização explícita é ilegal e antiético. Evidências completas do laboratório podem ser disponibilizadas a recrutadores e educadores mediante solicitação.**
</em>

### O que é ARP spoofing?

<div class="justify-text">

<p class="indent" style="text-indent: 2rem;">
ARP, sigla de Address Resolution Protocol, é o protocolo responsável por associar endereços IP a endereços MAC dentro de uma rede local. Os dispositivos mantêm uma tabela ARP, que funciona como um banco de associações entre esses dois tipos de endereço.
</p>

<p class="indent" style="text-indent: 2rem;">
ARP spoofing ocorre quando um atacante envia respostas ARP falsificadas, fazendo com que outros dispositivos armazenem associações incorretas entre IP e MAC em suas tabelas ARP. Essa técnica é frequentemente usada em ataques man-in-the-middle para interceptar, ler ou modificar tráfego entre dispositivos, sequestrar sessões, provocar negação de serviço, apoiar DNS poisoning e capturar credenciais ou outros dados.
</p>

<p class="indent" style="text-indent: 2rem;">
Este projeto demonstra ARP spoofing e apresenta formas de prevenção. Primeiro, mostro a comunicação normal entre sistemas na mesma rede local, com capturas de pacotes no Baseline A. Em seguida, demonstro o ARP spoofing no Baseline B. Por fim, apresento medidas de mitigação e proteção.
</p>

</div>

<p style="text-align:center">
<em>Este laboratório foi criado no Proxmox com quatro máquinas virtuais:</em>
</p>

<h4>Legenda OPSEC — rótulos substituem os endereços reais:</h4>

<ul class="opsec-list">
  <li><span class="label"><strong>WINDOWS10_VICTIM</strong></span><span class="arrow">→</span><span class="desc">alvo de teste Windows 10</span></li>
  <li><span class="label"><strong>KALI_ATTACKER</strong></span><span class="arrow">→</span><span class="desc">máquina Kali usada no teste controlado</span></li>
  <li><span class="label"><strong>DNS_SERVER</strong></span><span class="arrow">→</span><span class="desc">Raspberry Pi usado como DNS local</span></li>
  <li><span class="label"><strong>UBUNTU_SERVER</strong></span><span class="arrow">→</span><span class="desc">servidor Ubuntu com serviços web e de arquivos</span></li>
</ul>

---

### Baseline A — funcionamento normal, sem ARP spoofing

<p class="indent" style="text-indent: 2rem;">
O cliente Windows, <strong>WINDOWS10_VICTIM</strong>, acessa o endereço <code>ubuntu.lab</code>, hospedado em <strong>UBUNTU_SERVER</strong>. O cliente resolve o nome utilizando o resolvedor DNS local, <strong>DNS_SERVER</strong>, e depois estabelece conexão com o servidor.
</p>

<p class="indent" style="text-indent: 2rem;">
A captura abaixo mostra, no Wireshark, a resposta DNS enviada por <strong>DNS_SERVER</strong>, contendo um registro A que aponta corretamente para <strong>UBUNTU_SERVER</strong>.
</p>

Filtro utilizado:

```text
dns.qry.name == "ubuntu.lab"
```

![Baseline A - captura DNS no Wireshark](/assets/images/arp-spoofing/BASE_A_WIRESHARK_DNS.png)

<p class="indent" style="text-indent: 2rem;">
Quando <strong>WINDOWS10_VICTIM</strong> executa <code>nslookup ubuntu.lab</code>, a consulta é processada por <strong>DNS_SERVER</strong>, que retorna o endereço IP legítimo de <strong>UBUNTU_SERVER</strong>.
</p>

![Baseline A - nslookup de ubuntu.lab](/assets/images/arp-spoofing/BASE_A_WINDOWS_NSLOOKUP.png)

---

### Baseline B — ARP spoofing em um cenário man-in-the-middle

**Qual é o impacto real quando um atacante consegue realizar ARP spoofing dentro de uma rede?**

![Baseline B - topologia MITM](/assets/images/arp-spoofing/FINAL_BASE_B_SYSTEMS.png)

No cenário MITM, **KALI_ATTACKER** envenena as tabelas ARP dos dois endpoints para fazer com que o tráfego entre **WINDOWS10_VICTIM** e **UBUNTU_SERVER** passe pela máquina atacante.

<p class="indent" style="text-indent: 2rem;">
Todas as máquinas virtuais estavam conectadas à mesma bridge virtual dentro do Proxmox. Durante esta etapa, o ARP spoofing foi realizado de modo que tanto <strong>WINDOWS10_VICTIM</strong> quanto <strong>UBUNTU_SERVER</strong> passassem a associar o endereço MAC de <strong>KALI_ATTACKER</strong> aos endereços IP um do outro.
</p>

<p class="indent" style="text-indent: 2rem;">
Como resultado, o tráfego entre os dois hosts legítimos foi retransmitido de forma transparente por <strong>KALI_ATTACKER</strong>, criando um cenário man-in-the-middle.
</p>

<p class="indent" style="text-indent: 2rem;">
Para executar o teste de ARP spoofing em <strong>KALI_ATTACKER</strong>, usei a ferramenta <code>arpspoof</code>, fornecida pelo pacote <code>dsniff</code>.
</p>

Abra dois terminais separados na máquina Kali.

**Terminal 1 — alterar a tabela ARP da máquina Windows**

O objetivo é fazer o Windows associar o IP do servidor Ubuntu ao endereço MAC da máquina Kali.

```bash
arpspoof -i eth0 -t WINDOWS10_VICTIM_IP UBUNTU_SERVER_IP
```

Parâmetros:

- `-i eth0`: interface de rede usada pela máquina Kali. Ela pode ter outro nome, como `ens18`; confirme com `ip a`.
- `-t WINDOWS10_VICTIM_IP`: endereço IP do alvo Windows.
- `UBUNTU_SERVER_IP`: endereço IP do servidor que será representado falsamente para o alvo.

**Terminal 2 — alterar a tabela ARP do servidor Ubuntu**

O objetivo é fazer o servidor Ubuntu associar o IP da máquina Windows ao endereço MAC da Kali.

```bash
arpspoof -i eth0 -t UBUNTU_SERVER_IP WINDOWS10_VICTIM_IP
```

<p class="indent" style="text-indent: 2rem;">
Nesse ponto, o cenário MITM está ativo. O tráfego do Windows para o Ubuntu, e vice-versa, passa pela máquina Kali.
</p>

<p class="indent" style="text-indent: 2rem;">
Uma forma de validar isso é consultar a tabela ARP no Windows com <code>arp -a</code>. O endereço IP do servidor Ubuntu deve aparecer associado ao endereço MAC da máquina Kali.
</p>

![Baseline B - tabela ARP no Ubuntu](/assets/images/arp-spoofing/BASE_B_UBUNTU_ARP_2.png)

<p class="indent" style="text-indent: 2rem;">
A tabela ARP de <strong>WINDOWS10_VICTIM</strong> mostra o mesmo comportamento observado no servidor: os endereços IP do servidor e da máquina atacante aparecem associados ao endereço MAC de <strong>KALI_ATTACKER</strong>.
</p>

<p class="indent" style="text-indent: 2rem;">
Isso confirma que a vítima foi induzida a enviar para a máquina atacante os quadros que deveriam ser destinados ao servidor.
</p>

![Baseline B - tabela ARP no Windows](/assets/images/arp-spoofing/BASE_B_ARP_WINDOWS.png)

<p class="indent" style="text-indent: 2rem;">
Em seguida, foi possível observar, com <code>tcpdump</code> em <strong>KALI_ATTACKER</strong>, a requisição HTTP GET enviada por <strong>WINDOWS10_VICTIM</strong> e a resposta HTTP 200 retornada pelo servidor.
</p>

<p class="indent" style="text-indent: 2rem;">
Esses pacotes demonstram que a máquina atacante estava retransmitindo o tráfego entre a vítima e o servidor de forma transparente.
</p>

<p class="indent" style="text-indent: 2rem;">
Os cabeçalhos IP continuam mostrando <strong>WINDOWS10_VICTIM</strong> como origem e <strong>UBUNTU_SERVER</strong> como destino. Entretanto, os quadros Ethernet são recebidos pela máquina atacante. Isso confirma a interceptação e o encaminhamento do tráfego, preservando os endpoints IP e TCP originais.
</p>

![Baseline B - tcpdump na máquina Kali](/assets/images/arp-spoofing/BASE_B_KALI_ATTACKER_TCP_DUMP_ARP_SPOOFING.png)

<p class="indent" style="text-indent: 2rem;">
A captura no Wireshark também mostra uma mensagem ICMP Redirect observada durante o teste. Esse tipo de mensagem indica uma alteração sugerida no caminho de roteamento.
</p>

<p class="indent" style="text-indent: 2rem;">
Neste contexto, ela representa um efeito adicional na camada de rede causado pela mudança no caminho do tráfego. Esse tipo de evidência pode ajudar o analista a entender como a rede está percebendo a alteração da topologia em camada 2.
</p>

![Baseline B - captura no Wireshark](/assets/images/arp-spoofing/BASE_B_WIN10_WIRESHARK_ARP.png)

### Como detectar ARP spoofing

Alguns indicadores úteis incluem:

- alterações inesperadas na associação entre endereços IP e MAC;
- vários endereços IP associados ao mesmo MAC;
- mudanças frequentes na tabela ARP;
- tráfego ARP em volume incomum;
- respostas ARP não solicitadas;
- alertas de ferramentas como Arpwatch, XArp, Snort ou Suricata;
- comportamento anormal de roteamento ou mensagens ICMP Redirect vindas de hosts que não são gateways.

### Como prevenir ARP spoofing

**Controles em switches**

- habilitar **DHCP snooping**;
- habilitar **Dynamic ARP Inspection — DAI**;
- aplicar **port security**;
- limitar dispositivos permitidos por porta;
- monitorar mudanças inesperadas em bindings de IP e MAC.

**Entradas ARP estáticas**

Configurar associações estáticas entre IP e MAC para sistemas críticos, como gateways e servidores importantes.

Esse controle pode impedir alterações não autorizadas, embora não seja prático para redes grandes ou muito dinâmicas.

**Hardening em hosts Linux**

Ferramentas como **ArpON** e regras com **arptables** podem ajudar a validar ou bloquear atividades ARP suspeitas.

**Proteção em endpoints Windows**

Ferramentas como **XArp** podem detectar mudanças incomuns, bloquear tentativas e gerar alertas sobre possíveis ataques.

**Monitoramento de rede**

Serviços como **Arpwatch** e **Arpalert** podem identificar alterações inesperadas nas associações entre IP e MAC.

**IDS e IPS**

Soluções como **Snort** e **Suricata** podem detectar padrões anormais de tráfego ARP e encaminhar alertas para o SIEM.

**Criptografia e VPN**

O uso de **TLS/HTTPS**, criptografia ponta a ponta e VPNs não impede o ARP spoofing em si, mas reduz significativamente o valor do tráfego interceptado.

> Mesmo que o atacante consiga redirecionar o tráfego, a criptografia ajuda a proteger a confidencialidade e a integridade dos dados.

### Conclusão

Este laboratório demonstrou:

- o funcionamento normal da resolução e da comunicação na rede local;
- como associações ARP falsificadas podem redirecionar tráfego;
- como validar um cenário MITM por meio das tabelas ARP, `tcpdump` e Wireshark;
- quais evidências podem apoiar a investigação;
- quais controles em hosts, switches e ferramentas de monitoramento ajudam a reduzir o risco.
