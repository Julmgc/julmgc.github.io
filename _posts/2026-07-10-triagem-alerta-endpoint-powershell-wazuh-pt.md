---
title: "Triagem de alerta PowerShell com Sysmon e Wazuh"
date: 2026-07-10
layout: single
lang: pt-BR
translation_key: endpoint-alert-triage-powershell-wazuh
author_profile: true
author: julia_pt
sidebar:
  nav: "sidebar_pt"
toc: true
toc_sticky: true
categories: [Laboratórios, SOC]
tags:
  [
    Wazuh,
    Windows,
    Sysmon,
    PowerShell,
    Segurança-de-Endpoints,
    Engenharia-de-Detecção,
    Resposta-a-Incidentes,
    MITRE-ATT&CK,
  ]
excerpt: "Um laboratório de triagem de endpoint Windows com PowerShell, Sysmon e Wazuh, comparando eventos semelhantes e analisando em detalhe uma execução selecionada."
permalink: /pt/laboratorios/triagem-alerta-endpoint-powershell-wazuh/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Um fluxo de investigação de endpoint em que um sinal de PowerShell é analisado com Sysmon e Wazuh."
  image_height: 300px
---

### Visão geral

Neste laboratório, gerei quatro execuções seguras de PowerShell em um endpoint Windows 10 e usei **Sysmon + Wazuh** para revisar os eventos resultantes.

O objetivo foi comparar execuções semelhantes, selecionar uma delas para análise mais detalhada e examinar a evidência de criação de processo registrada pelo Sysmon.

O fluxo demonstrado foi:

> **gerar o sinal → revisar no Wazuh → selecionar o evento → analisar a evidência → documentar a conclusão**

### Arquitetura do laboratório

Usei uma estrutura pequena para monitoramento do endpoint Windows:

- **Wazuh** — manager, indexer e dashboard;
- **Windows 10** — endpoint monitorado;
- **Sysmon** — telemetria de criação de processos;
- **Wazuh Agent** — envio dos eventos do endpoint ao Wazuh.

Os eventos do Sysmon forneceram campos como linha de comando, usuário, host, processo pai e timestamp para análise.

### Execuções de PowerShell

Para praticar a triagem, gerei quatro comandos seguros de PowerShell em um intervalo curto:

```powershell
powershell.exe -Command "Get-Process | Select-Object -First 5"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
powershell.exe -Command "Get-Service | Select-Object -First 5"
powershell.exe -Command "whoami; hostname; Get-Date"
```

Todos eram inofensivos. O objetivo foi produzir eventos semelhantes e posteriormente comparar suas linhas de comando.

### Eventos no Wazuh

As execuções apareceram no Wazuh sob a mesma família de detecção relacionada ao PowerShell.

![Eventos relacionados ao PowerShell no Wazuh](/assets/images/proj-4/WIN-ps-initial-alert.png)

Os eventos foram comparados a partir da linha de comando registrada pelo Sysmon.

### Revisão da execução selecionada

Entre os quatro comandos, selecionei para análise mais detalhada:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
```

A combinação de `ExecutionPolicy Bypass` com um teste de conectividade para um host e uma porta específicos diferenciava esse evento dos demais.

O comando também pode ser usado em tarefas administrativas legítimas. Por isso, a análise não deve se limitar à linha de comando: é necessário verificar se o usuário e o host normalmente executam esse tipo de atividade e se o contexto observado é compatível com a função do endpoint.

Os outros comandos (`Get-Process`, `Get-Service` e `whoami; hostname; Get-Date`) eram verificações administrativas simples de processos, serviços e identidade do host.

### Evidência do Sysmon

Após selecionar a execução, abri no Wazuh o evento de criação de processo registrado pelo Sysmon.

![Evento de criação de processo registrado pelo Sysmon](/assets/images/proj-4/WIN-ps-raw-event.png)

O evento confirmou a linha de comando completa, o uso de `-NoProfile` e `-ExecutionPolicy Bypass`, além do host, usuário e timestamp associados.

Esses dados permitiram validar exatamente qual comando havia sido executado e em qual contexto.

### Conclusão

O evento selecionado se destacou pela combinação de `ExecutionPolicy Bypass` com um teste de conectividade para um host e uma porta específicos.

A evidência do Sysmon permitiu validar a linha de comando, o usuário, o host e o momento da execução, sem indicar impacto adicional no cenário analisado.

O laboratório demonstrou um fluxo simples de triagem de endpoint:

> **gerar o evento → identificar o sinal → revisar a evidência → contextualizar → decidir**
