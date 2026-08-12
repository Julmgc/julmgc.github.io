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
  image_description: "Fluxo de detecção, análise e contenção de atividades web em um laboratório com Wazuh."
  image_height: 300px
---

### Resumo do projeto

Montei um laboratório em **Proxmox** com **Wazuh, pfSense, Nginx/Flask e uma Jumpbox** para gerar e analisar tráfego web controlado.

Foram testados três cenários:

- falhas repetidas de autenticação;
- reconhecimento de endpoints web;
- requisições com padrão semelhante a SQL injection.

Também criei uma regra local no Wazuh para priorizar o sinal de SQL injection e validei uma contenção seletiva com **UFW**.

### Arquitetura do laboratório

| Componente    | Função                                            |
| ------------- | ------------------------------------------------- |
| **Wazuh**     | Manager, indexer e dashboard                      |
| **UbuntuApp** | Nginx, aplicação Flask e agente do Wazuh          |
| **pfSense**   | Gateway e firewall                                |
| **Jumpbox**   | Host controlado usado para gerar tráfego de teste |

A Jumpbox (`192.168.20.10`) e a UbuntuApp (`192.168.20.50`) estavam na mesma VLAN, portanto o tráfego HTTP entre elas não precisava atravessar o pfSense.

A aplicação Flask hospedada na UbuntuApp gerava logs JSON posteriormente ingeridos e pesquisados no Wazuh.

### Falhas repetidas de autenticação

Gerei aproximadamente 20 tentativas de login malsucedidas a partir da Jumpbox (`192.168.20.10`) contra o endpoint `/login` da aplicação Flask hospedada na UbuntuApp.

![Tentativas de login malsucedidas](/assets/images/proj-1/LOGIN-ATTEMPTS.png)

No Wazuh, a consulta retornou 20 eventos `POST /login` com status `401`, todos originados pela Jumpbox (`192.168.20.10`).

![Eventos de autenticação filtrados no Wazuh Discover](/assets/images/proj-1/WAZUH_COUNT_UP.png)

O sinal relevante não foi o status `401` isolado, mas a repetição das falhas pela mesma origem em uma janela curta.

Como melhoria da detecção, esse comportamento poderia ser tratado com um limiar temporal agrupado por `srcip`.

### Reconhecimento de recursos web

Também gerei, a partir da Jumpbox (`192.168.20.10`), requisições para diferentes endpoints da aplicação Flask, simulando uma sequência de reconhecimento web.

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
  or data.path:"/robots.txt"
)
```

![Eventos de reconhecimento filtrados no Wazuh Discover](/assets/images/proj-1/wazuh_discover_hits_2.png)

<p style="font-size: 0.85em; font-style: italic; text-align: center;">
A consulta no Wazuh retornou sete eventos no intervalo analisado, todos originados pela Jumpbox (<code>192.168.20.10</code>) e associados aos endpoints testados na aplicação Flask.
</p>

Embora as respostas tenham sido `404`, a sequência de requisições da mesma origem contra diferentes endpoints sensíveis tornou o comportamento mais relevante do que cada evento isolado.

Uma detecção mais robusta poderia considerar a quantidade de endpoints distintos, a taxa de requisições e origens previamente autorizadas para testes.

### Detecção de padrão semelhante a SQL injection

A partir da Jumpbox, enviei requisições de teste ao endpoint `/search` da aplicação Flask contendo padrões semelhantes a SQL injection, como:

```text
' OR 1=1 --
```

O objetivo foi gerar telemetria detectável no Wazuh, sem explorar a aplicação.

![Requisições de teste enviadas ao endpoint search](/assets/images/proj-1/payloads.png)

Para validar a ingestão, filtrei no Wazuh uma das strings enviadas.

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
A consulta retornou três eventos relacionados ao mesmo valor de <code>query_string</code>, todos originados pela Jumpbox durante a validação.
</p>

A consulta confirmou que o padrão enviado pela Jumpbox foi registrado na telemetria da aplicação e pôde ser identificado no Wazuh, fornecendo a evidência necessária para a etapa seguinte de priorização do alerta.

### Regra personalizada de priorização

O Wazuh possui uma regra nativa para padrões web relacionados a SQL injection:

```text
rule.id: 31164
level: 6
```

Para tornar o sinal mais visível durante o laboratório, criei uma regra local que gera um novo alerta a partir do disparo da regra nativa:

```text
Regra local: 100200
Nível: 12
Regra de origem: 31164
MITRE ATT&CK: T1190 — Exploit Public-Facing Application
```

![Regra 100200 no local_rules.xml](/assets/images/proj-1/RULE.ID-100200-local_rules.png)

A regra `100200` gerou alertas de nível `12`, facilitando sua priorização no Wazuh.

![Alerta da regra local 100200 com nível 12 no Wazuh Discover](/assets/images/proj-1/ruleID100200-BACKUP.png)

**Limitação da implementação**

A regra local demonstrada gerava um novo alerta para cada ocorrência da regra `31164`.

Uma versão mais robusta poderia aplicar correlação temporal por `data.srcip` e considerar frequência, comportamento esperado da aplicação e contexto da origem.

### Recursos de apoio à análise

Criei um dashboard no Wazuh para visualizar origem, horário e URLs associadas aos alertas.

![Dashboard de atividades web](/assets/images/proj-1/E4-SOC-WEB-ATTACKS.png)

Também desenvolvi um script Python para resumir IPs, endpoints, códigos de status e query strings.

<a href="https://github.com/Julmgc/detections/blob/main/helpers/summarize_flask_logs.py" target="_blank" rel="noopener noreferrer">
Script summarize_flask_logs.py
</a>

Exemplo de uso:

```bash
python3 summarize_flask_logs.py /var/log/flaskapp/app.log
```

![Resumo dos logs em Python](/assets/images/proj-1/python-triage-summary.png)

O script organiza os registros da aplicação por:

- principais IPs de origem;
- endpoints mais acessados;
- códigos de status;
- query strings observadas.

Esses recursos ajudam a estruturar a análise inicial, mas as conclusões continuam dependendo da revisão dos eventos no Wazuh e dos logs da aplicação Flask.

### Contenção e validação

Como a Jumpbox (`192.168.20.10`) e a UbuntuApp (`192.168.20.50`) estavam na mesma VLAN, apliquei a contenção diretamente na UbuntuApp com **UFW**.

A regra bloqueou conexões `TCP/80` originadas pela Jumpbox (`192.168.20.10`) em direção à aplicação web na UbuntuApp, mantendo o acesso SSH disponível para gerenciamento.

![Regra de contenção no UFW](/assets/images/proj-1/ufw-status.png)

Depois, validei a conectividade a partir da Jumpbox.

![Validação do bloqueio da porta 80 com Nmap e Netcat](/assets/images/proj-1/ufw-port-80-validation.png)

O Nmap mostrou `22/tcp` aberta para SSH e `80/tcp` como `filtered`. O teste com Netcat também resultou em timeout na porta 80.

Os resultados confirmaram que as conexões HTTP originadas pela Jumpbox estavam sendo bloqueadas sem remover o acesso de gerenciamento.

### Conclusão

O laboratório integrou logs estruturados da aplicação Flask, detecção no Wazuh, priorização de alertas, apoio à análise com dashboard e Python e contenção seletiva no endpoint.

A validação mostrou que o bloqueio aplicado no UFW filtrou `TCP/80` para a origem de teste enquanto preservava o acesso SSH.

Como melhorias futuras, as regras poderiam incorporar correlação temporal e contexto adicional da origem para reduzir ruído e tornar a priorização mais precisa.
