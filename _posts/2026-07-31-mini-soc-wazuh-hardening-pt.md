---
title: "Mini laboratório SOC: triagem web, detecção e contenção com Wazuh"
date: 2026-07-31
layout: single
lang: pt-BR
translation_key: mini-soc-wazuh-hardening
author_profile: true
author: julia_pt

sidebar:
  nav: "sidebar_pt"

toc: true
toc_sticky: true
categories: [Laboratórios, SOC]
tags: [Wazuh, SIEM, pfSense, Proxmox, Engenharia-de-Detecção]
excerpt: "Um miniambiente SOC em Proxmox com telemetria estruturada, detecção personalizada no Wazuh, triagem de atividades web suspeitas e contenção validada no endpoint."
permalink: /pt/laboratorios/mini-soc-wazuh-hardening/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Um fluxo completo de SOC, desde a geração do sinal até a detecção, triagem, contenção e validação."
  image_height: 300px
---

### Resumo executivo

---

Construí um miniambiente SOC em **Proxmox** com Wazuh, pfSense, uma aplicação Flask atrás do Nginx e uma Jumpbox utilizada para gerar tráfego controlado.

O laboratório analisou três tipos de atividade:

- falhas repetidas de autenticação;
- reconhecimento de endpoints web;
- requisições com padrões semelhantes a SQL injection.

No cenário de SQL injection, utilizei uma regra nativa do Wazuh e criei a regra local `100200` para aumentar a prioridade do alerta.

A contenção foi aplicada diretamente na UbuntuApp com UFW, bloqueando conexões `TCP/80` originadas pela Jumpbox e preservando o acesso SSH para gerenciamento.

O fluxo demonstrado foi:

> **geração do sinal → detecção → triagem → contenção → validação**

### Arquitetura do laboratório

---

O ambiente foi composto por quatro máquinas virtuais:

| Componente    | Função                                                |
| ------------- | ----------------------------------------------------- |
| **Wazuh**     | Manager, indexer e dashboard em instalação all-in-one |
| **UbuntuApp** | Nginx, aplicação Flask e agente do Wazuh              |
| **pfSense**   | Gateway, políticas de firewall e fonte de syslog      |
| **Jumpbox**   | Bastion e origem controlada do tráfego de teste       |

![Layout do laboratório no Proxmox](/assets/images/proj-1/PROXMOX-lab-layout.png)

**Topologia relevante**

O tráfego analisado foi gerado dentro da VLAN20:

A Jumpbox foi usada como origem das requisições, enquanto a UbuntuApp atuou como servidor web alvo.

```text
Jumpbox
192.168.20.10
      ↓
Tráfego HTTP dentro da VLAN20
      ↓
UbuntuApp
192.168.20.50
```

> **Observação de SOC:** como a Jumpbox e a UbuntuApp estavam na mesma VLAN, o tráfego lateral podia ocorrer sem atravessar o pfSense. Por isso, a telemetria do endpoint e da aplicação foi essencial para a análise e a contenção.

A aplicação Flask gerava logs JSON com os principais campos das requisições HTTP, posteriormente ingeridos e pesquisados no Wazuh.

### Falhas repetidas de autenticação

---

Gerei, a partir da Jumpbox, aproximadamente 20 tentativas de login malsucedidas contra o endpoint `/login`.

![Tentativas de login malsucedidas](/assets/images/proj-1/LOGIN-ATTEMPTS.png)

No Wazuh, a consulta retornou 20 eventos `POST /login` com status `401`, todos originados por `192.168.20.10`.

![Eventos de autenticação filtrados no Wazuh Discover](/assets/images/proj-1/WAZUH_COUNT_UP.png)

O sinal relevante não foi o status `401` isolado, mas a repetição das falhas pela mesma origem em uma janela curta. Em produção, a detecção deveria usar um limiar por `srcip`.

### Reconhecimento de recursos web

---

Também gerei requisições para diferentes endpoints sensíveis, simulando uma atividade de reconhecimento web.

![Rajada de requisições para endpoints sensíveis](/assets/images/proj-1/QUICK BURST.png)

Consulta DQL utilizada:

```text
agent.name:"ubuntu-app"
and data.app:"ticketing-lab"
and data.event:"http_request"
and data.srcip:"192.168.20.10"
and (
  data.path:"/wp-login.php"
  or data.path:"/admin"
  or data.path:"/.git/config"
  or data.path:"/phpmyadmin"
  or data.path:"/.env"
  or data.path:"/server-status"
)
```

![Eventos de reconhecimento filtrados no Wazuh Discover](/assets/images/proj-1/wazuh_discover_hits.png)

<p style="font-size: 0.85em; font-style: italic; text-align: center;">
A consulta confirmou que uma única origem acessou diferentes endpoints sensíveis em sequência.
</p>

Embora as respostas tenham sido 404, o padrão combinado é mais relevante do que cada evento isoladamente. Em produção, a detecção deveria considerar a quantidade de endpoints distintos, a taxa de requisições e scanners autorizados.

### Detecção de padrão semelhante a SQL injection

---

Enviei requisições de teste ao endpoint `/search` contendo padrões codificados para URL, como:

```text
' OR 1=1 --
```

O objetivo foi gerar telemetria detectável no Wazuh, sem explorar a aplicação.

![Requisições de teste enviadas ao endpoint search](/assets/images/proj-1/payloads.png)

Para validar a ingestão, filtrei no Wazuh uma das strings enviadas:

**Consulta DQL**

```text
agent.name:"ubuntu-app"
and data.app:"ticketing-lab"
and data.path:"/search"
and data.srcip:"192.168.20.10"
and data.query_string:"q=%27%20OR%201%3D1%20--"
```

![Sinal semelhante a SQL injection no Wazuh](/assets/images/proj-1/wazuh-SQLi.png)

<p style="font-size: 0.85em; font-style: italic; text-align: center;">
A consulta retornou três eventos relacionados ao mesmo valor de `query_string`, pois a mesma requisição foi enviada mais de uma vez durante a validação.
</p>

### Regra personalizada de priorização

---

O Wazuh possui uma regra nativa para padrões web relacionados a SQL injection:

```text
rule.id: 31164
level: 6
```

Para tornar o sinal mais visível durante a operação do laboratório, criei uma regra local que gera um novo alerta a partir do disparo da regra nativa.

```text
Regra local: 100200
Nível: 12
Regra de origem: 31164
MITRE ATT&CK: T1190 — Exploit Public-Facing Application
```

![Regra 100200 no local_rules.xml](/assets/images/proj-1/RULE.ID-100200-local_rules.png)

A regra local `100200` gerou alertas de nível `12`, tornando o sinal mais visível para triagem e priorização no Wazuh.

![Alerta da regra local 100200 com nível 12 no Wazuh Discover](/assets/images/proj-1/ruleID100200-BACKUP.png)

**Limitação da regra**

A versão demonstrada gerava um novo alerta para cada ocorrência da regra 31164.

Em produção, seria mais adequado aplicar um limiar temporal, agrupado por data.srcip, e considerar scanners autorizados, comportamento normal da aplicação e criticidade do ativo.

### Como seria a triagem em produção

---

Em produção, a triagem começaria pelo alerta da regra `100200`:

```text
agent.name:"ubuntu-app" and rule.id:"100200"
```

O analista validaria a origem, o endpoint, a query_string, o full_log e a frequência dos eventos.

Em seguida, correlacionaria a atividade com outros sinais da mesma origem, como falhas de autenticação, reconhecimento web e conexões de rede, além de verificar se havia autorização para testes.

O objetivo seria determinar se o alerta representa atividade legítima, uma tentativa isolada ou parte de uma sequência ofensiva mais ampla.

### Recursos de apoio à triagem

---

Criei um dashboard no Wazuh para visualizar origem, horário e URLs associadas aos alertas.

![Dashboard de ataques web](/assets/images/proj-1/E4-SOC-WEB-ATTACKS.png)

Também desenvolvi um script Python para resumir IPs, endpoints, códigos de status e query strings.

<a href="https://github.com/Julmgc/detections/blob/main/helpers/summarize_flask_logs.py" target="_blank" rel="noopener noreferrer">
Script summarize_flask_logs.py
</a>

Exemplo de uso:

```bash
python3 summarize_flask_logs.py /var/log/flaskapp/app.log
```

![Resumo de triagem em Python](/assets/images/proj-1/python-triage-summary.png)

O script resume os registros da aplicação por:

- principais IPs de origem;
- endpoints mais acessados;
- códigos de status;
- query strings observadas.

Esses recursos apoiam a análise inicial, mas não substituem a validação dos eventos no Wazuh e nos logs da aplicação.

### Contenção e validação

---

Como a Jumpbox e a UbuntuApp estavam na mesma VLAN, as requisições de teste eram enviadas diretamente entre as duas máquinas, sem atravessar o pfSense.

Por isso, apliquei a contenção diretamente na UbuntuApp com UFW, bloqueando conexões `TCP/80` originadas por `192.168.20.10`.

Mantive o acesso SSH liberado para o gerenciamento das máquinas virtuais.

![Regra de contenção no UFW](/assets/images/proj-1/ufw-status.png)

Após aplicar a regra no UFW, validei a conectividade a partir da Jumpbox.

![Validação do bloqueio da porta 80 com Nmap e Netcat](/assets/images/proj-1/ufw-port-80-validation.png)

O Nmap mostrou que a porta `22/tcp` permanecia aberta para SSH, enquanto a porta `80/tcp` aparecia como `filtered`. O teste adicional com Netcat também resultou em timeout na porta 80.

Esses resultados confirmaram que o acesso de gerenciamento foi preservado e que as conexões HTTP originadas pela Jumpbox estavam sendo bloqueadas.

### Conclusão

---

O laboratório integrou telemetria estruturada, detecção no Wazuh, análise do alerta e contenção seletiva no endpoint.

A validação mostrou que a porta `80/tcp` foi filtrada para a Jumpbox, enquanto o acesso SSH permaneceu disponível para gerenciamento.

Como próximos passos, a aplicação poderia incorporar consultas parametrizadas, validação de entrada, HTTPS, privilégio mínimo e correlação temporal dos alertas.
