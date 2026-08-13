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
excerpt: "Laboratório prático de triagem de endpoint com Sysmon e Wazuh, comparando execuções de PowerShell, analisando telemetria de criação de processos e contextualizando atividade potencialmente suspeita."
permalink: /pt/laboratorios/triagem-alerta-endpoint-powershell-wazuh/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Um fluxo de investigação de endpoint em que um sinal de PowerShell é analisado com Sysmon e Wazuh."
  image_height: 300px
---

---

### Resumo do projeto

Neste laboratório, gerei quatro execuções seguras de PowerShell em um endpoint **Windows 10** e usei **Sysmon + Wazuh** para comparar os eventos e revisar a telemetria de criação de processos.

### Execuções de PowerShell

Gerei quatro comandos seguros de PowerShell em um intervalo curto:

```powershell
powershell.exe -Command "Get-Process | Select-Object -First 5"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
powershell.exe -Command "Get-Service | Select-Object -First 5"
powershell.exe -Command "whoami; hostname; Get-Date"
```

### Eventos no Wazuh

As quatro execuções apareceram no Wazuh sob a mesma família de detecção relacionada ao PowerShell.

![Eventos de PowerShell no Wazuh](/assets/images/proj-4/WIN-ps-initial-alert.png)

A comparação foi feita a partir das linhas de comando registradas pelo Sysmon.

### Evento selecionado para análise

Selecionei este evento para uma revisão mais detalhada:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
```

A execução se destacou por combinar `ExecutionPolicy Bypass` com uma tentativa de conexão para `192.168.20.55:1514`, enquanto os demais comandos eram consultas administrativas simples.

`ExecutionPolicy Bypass` aumentou o interesse do evento, mas não foi tratado isoladamente como evidência de atividade maliciosa. A revisão considerou a linha de comando, usuário, host, horário e destino da conexão antes de contextualizar a execução.

### Evidência do Sysmon

No Wazuh, revisei o evento de criação de processo registrado pelo Sysmon.

![Evidência de criação de processo registrada pelo Sysmon](/assets/images/proj-4/WIN-ps-raw-event.png)

O evento registrou a linha de comando completa, incluindo `-NoProfile` e `-ExecutionPolicy Bypass`, além do usuário, host e timestamp associados.

Esses dados permitiram confirmar qual comando havia sido executado e em qual contexto.

### Conclusão

A comparação dos quatro eventos permitiu identificar uma execução que merecia revisão adicional pelo uso de `ExecutionPolicy Bypass` combinado com uma conexão para `192.168.20.55:1514`.

A evidência do Sysmon permitiu validar o comando e seu contexto sem indicar atividade adicional no cenário analisado.

> **gerar o sinal → comparar eventos → revisar a evidência → contextualizar → decidir**
