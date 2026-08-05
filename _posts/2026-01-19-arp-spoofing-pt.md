---
title: "ARP Spoofing: demonstração de ataque MITM e estratégias de mitigação"
date: 2026-01-19
layout: single
lang: pt-BR
translation_key: arp-spoofing
categories: [Laboratórios]
tags: [ARP, Spoofing, MITM, Wireshark]
excerpt: "Demonstração de ARP spoofing em um cenário man-in-the-middle: baseline, interceptação de tráfego, evidências e medidas de mitigação."
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
---

<em>
<strong>Importante:</strong> esta técnica é apresentada exclusivamente para fins educacionais e foi executada em um ambiente de laboratório privado e controlado. Realizar ARP spoofing em uma rede sem autorização explícita é ilegal e antiético. Evidências completas do laboratório podem ser disponibilizadas a recrutadores e educadores mediante solicitação.
</em>

### Visão geral

---

Este projeto demonstra como o **ARP spoofing** pode alterar a confiança entre dispositivos em uma rede local e criar uma posição **man-in-the-middle — MITM**.

Primeiro, documentei o comportamento normal da comunicação entre os sistemas. Em seguida, usei uma máquina Kali para enviar respostas ARP falsificadas aos dois endpoints, fazendo com que o tráfego entre uma vítima Windows e um servidor Ubuntu passasse pela máquina atacante.

O objetivo do laboratório foi demonstrar:

1. o funcionamento normal da resolução DNS e da comunicação local;
2. a alteração das associações entre endereços IP e MAC;
3. a criação de uma posição MITM em camada 2;
4. a validação da interceptação por meio das tabelas ARP, `tcpdump` e Wireshark;
5. os indicadores que podem ajudar em uma investigação;
6. as medidas de prevenção, limitação de impacto e monitoramento.

> Este laboratório se concentra na criação e validação da posição man-in-the-middle em camada 2. Em um projeto separado, utilizo essa posição para interceptar consultas DNS e injetar respostas falsificadas.
>
> [Ler o laboratório de DNS Spoofing via MITM baseado em ARP](/pt/laboratorios/dns-spoofing/){: target="\_blank" rel="noopener noreferrer" }

### O que é ARP?

---

ARP significa **Address Resolution Protocol**. É o protocolo utilizado para associar endereços IPv4 a endereços MAC dentro de uma rede local.

Quando um dispositivo precisa enviar dados para outro equipamento no mesmo segmento de rede, ele precisa descobrir qual endereço MAC corresponde ao endereço IP de destino.

O resultado é armazenado temporariamente em uma tabela ARP:

```text
Endereço IP → Endereço MAC
```

Exemplo:

```text
192.168.20.45 → 00:11:22:33:44:55
```

O ARP não possui um mecanismo nativo robusto de autenticação. Por isso, um dispositivo pode aceitar respostas ARP mesmo sem ter solicitado aquela informação, dependendo do sistema operacional e da implementação.

### O que é ARP spoofing?

---

ARP spoofing ocorre quando um atacante envia respostas ARP falsificadas, fazendo com que outros dispositivos armazenem associações incorretas entre IP e MAC em suas tabelas ARP.

Em um cenário man-in-the-middle, o atacante envia mensagens diferentes aos dois endpoints:

```text
Para a vítima:
“O IP do servidor está no meu endereço MAC.”

Para o servidor:
“O IP da vítima está no meu endereço MAC.”
```

Como resultado, os dois dispositivos enviam seus quadros Ethernet para o atacante.

A técnica pode ser usada para:

- interceptar tráfego;
- observar comunicações não criptografadas;
- modificar ou bloquear pacotes;
- provocar negação de serviço;
- apoiar sequestro de sessão;
- estabelecer uma posição para ataques posteriores, como DNS response spoofing.

A cadeia demonstrada neste laboratório é:

```text
Respostas ARP falsificadas
      ↓
Associações IP–MAC incorretas
      ↓
Tráfego enviado para a máquina Kali
      ↓
Kali retransmite os pacotes
      ↓
Posição man-in-the-middle estabelecida
```

O ARP spoofing cria a posição de interceptação, mas não significa automaticamente que todo o conteúdo poderá ser lido. Protocolos com criptografia e autenticação, como HTTPS e SSH, continuam protegendo o conteúdo da comunicação quando configurados e validados corretamente.

### Estrutura do laboratório

---

Este laboratório foi criado no **Proxmox**, utilizando quatro sistemas conectados à mesma bridge virtual.

| Identificador        | Função             | Descrição                                 |
| -------------------- | ------------------ | ----------------------------------------- |
| **WINDOWS10_VICTIM** | Alvo de teste      | Cliente Windows 10                        |
| **KALI_ATTACKER**    | Máquina atacante   | Executa ARP spoofing e captura de pacotes |
| **DNS_SERVER**       | Servidor DNS local | Raspberry Pi executando o resolvedor DNS  |
| **UBUNTU_SERVER**    | Servidor legítimo  | Hospeda serviços web e de arquivos        |

**Identificadores OPSEC**

Os seguintes nomes substituem os endereços reais utilizados no laboratório:

```text
WINDOWS10_VICTIM_IP
KALI_ATTACKER_IP
DNS_SERVER_IP
UBUNTU_SERVER_IP
```

Todos os testes foram realizados em uma rede privada e isolada.

### Baseline A — funcionamento normal, sem ARP spoofing

---

Embora este laboratório seja focado em ARP spoofing, a resolução DNS foi utilizada como parte do baseline para documentar o fluxo normal entre os sistemas antes da interceptação.

Nesta etapa, nenhuma resposta DNS foi falsificada.

O cliente Windows, `WINDOWS10_VICTIM`, acessava o hostname:

```text
ubuntu.lab
```

Esse hostname estava associado ao servidor `UBUNTU_SERVER`.

O cliente primeiro consultava o resolvedor DNS local, `DNS_SERVER`, e depois estabelecia a comunicação com o endereço IP legítimo retornado.

O fluxo esperado era:

```text
WINDOWS10_VICTIM
        ↓ consulta DNS
DNS_SERVER
        ↓ resposta legítima
WINDOWS10_VICTIM
        ↓ conexão
UBUNTU_SERVER
```

A captura abaixo mostra, no Wireshark, a resposta DNS enviada por `DNS_SERVER`, contendo um registro A que aponta corretamente para `UBUNTU_SERVER`.

Filtro utilizado:

```text
dns.qry.name == "ubuntu.lab"
```

![Baseline A — captura DNS no Wireshark](/assets/images/arp-spoofing/BASE_A_WIRESHARK_DNS.png)

Na máquina Windows, executei:

```powershell
nslookup ubuntu.lab
```

A consulta foi processada pelo servidor DNS local, que retornou o endereço IP legítimo do servidor Ubuntu.

![Baseline A — nslookup de ubuntu.lab](/assets/images/arp-spoofing/BASE_A_WINDOWS_NSLOOKUP.png)

Esse baseline confirmou que, antes do ataque:

- a resolução DNS funcionava corretamente;
- o hostname apontava para o servidor esperado;
- não havia evidências de manipulação nas tabelas ARP;
- a comunicação ocorria diretamente entre a vítima e o servidor.

### Baseline B — ARP spoofing em um cenário man-in-the-middle

---

**Topologia do ataque**

A máquina `KALI_ATTACKER` foi posicionada entre `WINDOWS10_VICTIM` e `UBUNTU_SERVER`.

![Baseline B — topologia MITM](/assets/images/arp-spoofing/FINAL_BASE_B_SYSTEMS.png)

Antes do ataque, a comunicação ocorria diretamente:

```text
WINDOWS10_VICTIM ←→ UBUNTU_SERVER
```

Depois do envenenamento das tabelas ARP:

```text
WINDOWS10_VICTIM
        ↓
KALI_ATTACKER
        ↓
UBUNTU_SERVER
```

A máquina Kali passou a receber os quadros Ethernet destinados ao outro endpoint e a retransmiti-los para o destino legítimo.

Os cabeçalhos IP continuavam identificando a vítima e o servidor como origem e destino. A alteração ocorreu nas associações utilizadas na camada 2.

### Execução do ARP spoofing

---

Para realizar o teste, usei a ferramenta `arpspoof`, fornecida pelo pacote `dsniff`.

Foram necessários dois terminais na máquina Kali, pois os dois endpoints precisavam receber informações ARP falsificadas.

**Terminal 1 — alterar a tabela ARP da máquina Windows**

O primeiro comando faz o Windows associar o endereço IP do servidor Ubuntu ao endereço MAC da máquina Kali:

```bash
sudo arpspoof \
  -i eth0 \
  -t WINDOWS10_VICTIM_IP \
  UBUNTU_SERVER_IP
```

Parâmetros:

- `-i eth0`: interface de rede utilizada pela máquina Kali;
- `-t WINDOWS10_VICTIM_IP`: endereço IP do alvo que receberá as respostas;
- `UBUNTU_SERVER_IP`: endereço IP que a Kali representará para a vítima.

A interface pode ter outro nome, como `ens18`. Ela pode ser identificada com:

```bash
ip a
```

Conceitualmente, a mensagem enviada para o Windows é:

```text
UBUNTU_SERVER_IP está associado ao MAC de KALI_ATTACKER
```

**Terminal 2 — alterar a tabela ARP do servidor Ubuntu**

O segundo comando faz o servidor Ubuntu associar o endereço IP da máquina Windows ao endereço MAC da Kali:

```bash
sudo arpspoof \
  -i eth0 \
  -t UBUNTU_SERVER_IP \
  WINDOWS10_VICTIM_IP
```

Conceitualmente, a mensagem enviada para o servidor é:

```text
WINDOWS10_VICTIM_IP está associado ao MAC de KALI_ATTACKER
```

Depois que os dois endpoints armazenam as associações falsas, o tráfego passa pela máquina atacante.

### Validação por meio das tabelas ARP

---

Na máquina Windows, a tabela ARP pode ser consultada com:

```powershell
arp -a
```

Na máquina Ubuntu, pode-se usar:

```bash
ip neigh
```

ou:

```bash
arp -n
```

A tabela ARP do Ubuntu mostrou o endereço IP da máquina Windows associado ao endereço MAC da Kali.

![Baseline B — tabela ARP no Ubuntu](/assets/images/arp-spoofing/BASE_B_UBUNTU_ARP_2.png)

A tabela ARP de `WINDOWS10_VICTIM` apresentou comportamento correspondente: o endereço IP do servidor Ubuntu aparecia associado ao MAC de `KALI_ATTACKER`.

![Baseline B — tabela ARP no Windows](/assets/images/arp-spoofing/BASE_B_ARP_WINDOWS.png)

Essas associações confirmam que:

```text
A vítima acredita que o servidor está no MAC da Kali
e
O servidor acredita que a vítima está no MAC da Kali
```

Como resultado, os dois endpoints passam a enviar para a máquina atacante quadros que deveriam ser entregues diretamente um ao outro.

### Validação da interceptação com tcpdump

---

Depois de estabelecer a posição MITM, observei o tráfego na interface da máquina Kali usando `tcpdump`.

A captura mostrou:

- a requisição HTTP GET enviada por `WINDOWS10_VICTIM`;
- a resposta HTTP 200 retornada por `UBUNTU_SERVER`;
- os pacotes atravessando a interface de `KALI_ATTACKER`.

![Baseline B — tcpdump na máquina Kali](/assets/images/arp-spoofing/BASE_B_KALI_ATTACKER_TCP_DUMP_ARP_SPOOFING.png)

Os cabeçalhos IP ainda apresentavam:

```text
Origem: WINDOWS10_VICTIM
Destino: UBUNTU_SERVER
```

Entretanto, os quadros Ethernet eram entregues primeiro à máquina Kali.

Essa diferença ocorre porque:

- IP pertence à camada 3;
- MAC e ARP operam na camada 2;
- o atacante altera o próximo destino Ethernet;
- os endpoints IP originais permanecem os mesmos.

A captura demonstrou que a máquina Kali não estava apenas observando pacotes enviados em broadcast. Ela estava efetivamente participando do caminho entre os dois endpoints.

Como o tráfego utilizado na demonstração era HTTP, seu conteúdo podia ser observado sem criptografia. Em uma sessão HTTPS ou SSH corretamente validada, o atacante ainda poderia observar metadados e bloquear a comunicação, mas não deveria conseguir ler facilmente o conteúdo.

### ICMP Redirect observado durante o teste

---

A captura no Wireshark também mostrou uma mensagem ICMP Redirect.

![Baseline B — captura no Wireshark](/assets/images/arp-spoofing/BASE_B_WIN10_WIRESHARK_ARP.png)

Mensagens ICMP Redirect normalmente são enviadas por um roteador para informar a um host que existe uma rota mais apropriada para determinado destino.

Neste laboratório, essa mensagem deve ser tratada como uma **anomalia de rede complementar**, e não como prova isolada de ARP spoofing.

Durante uma investigação, ela deveria ser correlacionada com evidências mais fortes, como:

- alterações inesperadas nas tabelas ARP;
- vários endereços IP associados ao mesmo MAC;
- tráfego passando por um endpoint que não deveria atuar como roteador;
- pacotes observados na interface da máquina intermediária;
- mensagens ICMP Redirect enviadas por um host não autorizado.

A presença de ICMP Redirect pode indicar comportamento anormal de roteamento ou configuração, mas não comprova sozinha que houve envenenamento ARP.

### Evidências principais do ataque

---

As evidências mais relevantes deste laboratório foram:

1. O MAC de `KALI_ATTACKER` apareceu associado ao IP do servidor na tabela ARP da vítima.
2. O MAC de `KALI_ATTACKER` apareceu associado ao IP da vítima na tabela ARP do servidor.
3. A máquina Kali observou o tráfego entre os dois endpoints.
4. A requisição HTTP e a resposta do servidor passaram pela interface da Kali.
5. Os endpoints IP permaneceram os mesmos, enquanto o caminho em camada 2 foi alterado.
6. O tráfego continuou funcionando porque a Kali retransmitia os pacotes.

A combinação dessas evidências confirma a criação da posição man-in-the-middle.

### Como detectar ARP spoofing

---

Um único evento ARP nem sempre é suficiente para concluir que existe um ataque. A análise deve considerar o contexto e a correlação entre diferentes fontes de dados.

**Indicadores nas tabelas ARP**

- alteração inesperada em uma associação IP–MAC;
- endereço IP do gateway associado a um MAC desconhecido;
- vários endereços IP associados ao mesmo MAC;
- mudanças repetidas na associação de um mesmo endereço IP;
- diferenças entre a tabela ARP atual e um baseline conhecido.

**Indicadores no tráfego**

- grande volume de respostas ARP;
- respostas ARP não solicitadas;
- respostas conflitantes para o mesmo endereço IP;
- um endpoint comum anunciando endereços pertencentes a outros dispositivos;
- mensagens ICMP Redirect originadas por um host que não deveria atuar como roteador.

**Indicadores comportamentais**

- interrupções ou instabilidade na comunicação;
- aumento de latência;
- conexões passando por um caminho inesperado;
- alertas de certificado em serviços normalmente confiáveis;
- conteúdo HTTP observado em um endpoint intermediário;
- conflitos frequentes entre IP e MAC.

**Fontes de dados úteis**

- tabelas ARP dos endpoints;
- capturas com Wireshark ou `tcpdump`;
- logs de switches gerenciáveis;
- logs de DHCP;
- alertas de IDS ou IPS;
- ferramentas de monitoramento ARP;
- telemetria de EDR;
- logs de firewall e SIEM.

**Ferramentas de detecção**

Algumas opções incluem:

- Arpwatch;
- Arpalert;
- XArp;
- Suricata;
- Snort;
- Zeek;
- Wireshark;
- ferramentas de monitoramento de switches.

Essas ferramentas podem identificar alterações suspeitas, mas os alertas devem ser correlacionados com mudanças legítimas, como renovação de equipamentos, failover ou reconfiguração da rede.

### Como prevenir ARP spoofing

---

A proteção mais efetiva combina controles no switch, nos endpoints, na arquitetura da rede e nos protocolos utilizados pelas aplicações.

**DHCP snooping**

O DHCP snooping cria uma tabela confiável com associações entre:

- endereço IP;
- endereço MAC;
- VLAN;
- porta do switch;
- concessão DHCP.

Essa tabela pode ser utilizada por outros controles, especialmente pelo Dynamic ARP Inspection.

O DHCP snooping não bloqueia sozinho todas as formas de ARP spoofing, mas fornece uma base de confiança para validar as mensagens ARP.

**Dynamic ARP Inspection**

O **Dynamic ARP Inspection — DAI** valida pacotes ARP utilizando associações confiáveis, normalmente obtidas pela tabela de DHCP snooping.

Quando uma mensagem ARP apresenta uma associação incompatível, o switch pode descartá-la antes que chegue aos endpoints.

O DAI é uma das medidas mais diretamente relacionadas à prevenção de ARP spoofing.

Limitações:

- exige switch gerenciável compatível;
- deve ser configurado corretamente por VLAN;
- dispositivos com IP estático podem exigir entradas adicionais;
- uma configuração incorreta pode bloquear tráfego legítimo.

**Port security**

Port security limita quais endereços MAC podem utilizar cada porta física do switch.

Esse controle ajuda a:

- impedir dispositivos não autorizados;
- limitar a quantidade de MACs por porta;
- bloquear ou alertar quando ocorre uma violação.

Port security não impede necessariamente um ataque iniciado por um dispositivo que já esteja autorizado, mas reduz a superfície de acesso à rede.

**Segmentação por VLAN e regras de firewall**

Separar sistemas em VLANs diferentes reduz o domínio de broadcast no qual as mensagens ARP circulam.

Como o ARP spoofing opera dentro do mesmo segmento de camada 2, a segmentação limita o número de dispositivos que podem ser diretamente afetados.

Exemplo:

```text
VLAN 10 — estações de trabalho
VLAN 20 — servidores
VLAN 30 — dispositivos de administração
VLAN 40 — equipamentos IoT
```

Regras de firewall e ACLs devem controlar a comunicação entre esses segmentos.

A segmentação possui uma limitação importante:

> VLANs não impedem ARP spoofing entre dispositivos que permanecem dentro da mesma VLAN.

**Entradas ARP estáticas**

Associações estáticas podem ser configuradas para sistemas críticos, como:

- gateways;
- servidores importantes;
- equipamentos de infraestrutura.

Isso impede que a associação seja alterada dinamicamente por respostas ARP falsas.

Entretanto, a abordagem não é prática para redes grandes ou ambientes que mudam frequentemente, pois exige manutenção manual.

**Hardening em hosts Linux**

Em sistemas Linux, ferramentas ou regras locais podem ajudar a detectar e bloquear atividade ARP suspeita.

Exemplos:

- ArpON;
- Arpwatch;
- Arpalert;
- regras de filtragem na camada Ethernet;
- monitoramento de mudanças em `ip neigh`.

Esses controles dependem da distribuição, do kernel e da arquitetura da rede.

**Proteção em endpoints Windows**

Ferramentas como XArp podem:

- monitorar alterações nas tabelas ARP;
- detectar associações suspeitas;
- gerar alertas;
- bloquear determinados comportamentos.

EDRs também podem ajudar quando o ataque é acompanhado por outras atividades no endpoint, embora nem todos ofereçam visibilidade detalhada sobre ARP.

**IDS e IPS**

Ferramentas como Snort e Suricata podem detectar padrões anormais, como:

- respostas ARP em excesso;
- associações conflitantes;
- alterações frequentes;
- tráfego originado por hosts inesperados.

Os alertas podem ser enviados para um SIEM e correlacionados com:

- logs DHCP;
- eventos do switch;
- alertas DNS;
- conexões de rede;
- relatos de usuários.

**Criptografia e VPN**

O uso de criptografia não impede que o atacante altere as tabelas ARP ou se posicione entre os dispositivos.

Entretanto, protocolos autenticados e criptografados reduzem significativamente o valor do tráfego interceptado.

Exemplos:

- HTTPS;
- SSH;
- SFTP;
- RDP com TLS e NLA;
- SMB Encryption;
- IPsec;
- WireGuard;
- outras VPNs.

Mesmo que o atacante consiga encaminhar os pacotes, a criptografia protege a confidencialidade e a integridade do conteúdo.

O atacante ainda pode:

- observar endereços IP e portas;
- analisar volume e horário das conexões;
- provocar indisponibilidade;
- interromper ou degradar a comunicação.

### Defesa em profundidade

---

Esses controles funcionam melhor quando combinados:

```text
DHCP snooping cria associações confiáveis entre IP, MAC, VLAN e porta
      ↓
Dynamic ARP Inspection valida e bloqueia mensagens ARP falsas
      ↓
Port security restringe dispositivos permitidos
      ↓
VLANs e firewall limitam o alcance do ataque
      ↓
HTTPS, SSH e VPNs protegem o conteúdo interceptado
      ↓
Monitoramento identifica alterações e comportamentos suspeitos
```

Nenhuma dessas medidas elimina o risco sozinha.

Os controles de switch tentam impedir o ataque. A segmentação limita seu alcance. A criptografia protege os dados mesmo quando ocorre interceptação. O monitoramento fornece visibilidade quando os controles preventivos falham.

### Resposta a um possível incidente

---

Quando houver suspeita de ARP spoofing:

1. Registrar as tabelas ARP antes de realizar alterações.
2. Capturar o tráfego ARP relevante.
3. Identificar quais IPs aparecem associados ao mesmo MAC.
4. Consultar os registros DHCP e as tabelas do switch.
5. Verificar em qual porta o MAC suspeito foi observado.
6. Isolar o dispositivo suspeito.
7. Limpar as entradas ARP afetadas.
8. Verificar se houve interceptação de tráfego não criptografado.
9. Investigar alertas de certificado ou redirecionamentos.
10. Monitorar se as associações falsas reaparecem.

A análise também deve verificar se o atacante realizou atividades posteriores, como:

- DNS spoofing;
- captura de credenciais;
- sequestro de sessão;
- alteração de tráfego;
- negação de serviço;
- reconhecimento de outros sistemas.

### Limpeza do laboratório

---

Depois do teste autorizado, os processos de ARP spoofing devem ser encerrados na máquina Kali:

```bash
sudo pkill arpspoof
```

Também é importante confirmar que não permaneceram regras temporárias de encaminhamento ou firewall.

Nos endpoints, as entradas ARP podem expirar naturalmente ou ser removidas conforme o sistema operacional.

No Windows, uma opção é:

```powershell
arp -d *
```

No Linux, pode-se limpar a tabela de vizinhos com:

```bash
sudo ip neigh flush all
```

Depois, valide novamente as associações:

```powershell
arp -a
```

```bash
ip neigh
```

Por fim, confirme que:

- o IP do servidor voltou a estar associado ao MAC legítimo;
- o IP da vítima voltou a estar associado ao MAC legítimo;
- a comunicação não passa mais pela Kali;
- a resolução DNS e os serviços funcionam normalmente.

### Principais conclusões

---

Este laboratório demonstrou que:

- ARP é responsável por associar endereços IP e MAC em uma rede local;
- respostas ARP falsificadas podem modificar essas associações;
- o envenenamento dos dois endpoints permite criar uma posição MITM;
- a alteração ocorre na camada 2, enquanto os endpoints IP permanecem os mesmos;
- tabelas ARP, `tcpdump` e Wireshark fornecem evidências úteis;
- tráfego não criptografado pode ser observado a partir da posição intermediária;
- criptografia reduz o impacto, mas não impede o ARP spoofing;
- controles de switch, segmentação e monitoramento devem ser combinados.

### Conclusão

---

Este laboratório demonstrou como respostas ARP falsificadas podem alterar a confiança entre dispositivos de uma rede local e fazer com que o tráfego entre dois endpoints passe por uma máquina intermediária.

A posição man-in-the-middle foi validada por meio:

- das associações incorretas nas tabelas ARP;
- do endereço MAC da Kali associado aos IPs legítimos;
- da observação do tráfego na interface da máquina atacante;
- da captura da requisição HTTP e da resposta do servidor;
- da preservação dos endpoints IP durante a alteração do caminho em camada 2.

O ARP spoofing demonstrado neste post não alterou as respostas DNS. Ele estabeleceu a posição de interceptação necessária para que outros ataques pudessem ser realizados.

No laboratório seguinte, essa posição MITM é utilizada para observar consultas DNS, injetar uma resposta falsificada e redirecionar a vítima para um endereço controlado pela máquina atacante:

[DNS Spoofing via MITM baseado em ARP](/pt/laboratorios/dns-spoofing/){: target="\_blank" rel="noopener noreferrer" }
