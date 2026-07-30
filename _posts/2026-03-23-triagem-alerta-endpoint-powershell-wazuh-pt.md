---
title: "Triagem de alertas de endpoint: investigando um sinal de PowerShell gerado em laboratório com Sysmon + Wazuh"
date: 2026-03-23
layout: single
lang: pt-BR
translation_key: endpoint-alert-triage-powershell-wazuh
author_profile: true
author: julia_pt
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
excerpt: "Um caso de triagem de endpoint Windows: gerar um sinal de PowerShell que exige análise, investigá-lo no Wazuh com evidências do Sysmon, compará-lo com atividades próximas no endpoint e documentar a conclusão do analista."
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

Alertas de endpoint no Windows costumam ser ambíguos. Um evento de PowerShell pode ser completamente benigno, levemente suspeito ou exigir escalonamento, dependendo da linha de comando, do processo pai, do contexto do host e das atividades observadas no mesmo período. Este projeto se concentra justamente nessa etapa de julgamento do analista.

Neste laboratório, eu:

- gerei um pequeno conjunto de execuções seguras de PowerShell em um endpoint Windows 10;
- usei **Sysmon + Wazuh** para revisar os eventos resultantes;
- identifiquei qual execução exigia uma análise mais detalhada;
- investiguei no Wazuh a evidência bruta de criação de processo;
- comparei o evento analisado com atividades próximas no mesmo endpoint;
- documentei uma conclusão prática de analista, sem presumir que contenção fosse necessária.

**Evidências produzidas:** visualizações de eventos no Wazuh, dados brutos de processos coletados pelo Sysmon, contexto de atividades próximas no host e uma breve conclusão analítica.

### Por que este cenário é realista para um SOC

- O sinal é sustentado por **telemetria real de criação de processos do Sysmon**, e não apenas por uma captura de tela do terminal.
- A mesma família ampla de regras do Wazuh pode corresponder a diferentes execuções de PowerShell. Por isso, o analista precisa examinar os **detalhes do evento original**, e não apenas o nome do alerta.
- O resultado não é simplesmente “bloqueei uma ameaça”. O resultado é uma **decisão clara do analista**: revisar, explicar, monitorar ou escalonar, conforme o contexto.

### Arquitetura do laboratório

Usei uma estrutura pequena, voltada ao monitoramento de endpoints Windows:

- **Wazuh (SIEM)** — manager, indexer e dashboard;
- **Endpoint Windows 10** — com Sysmon e agente do Wazuh instalados;
- **Sysmon** — coleta de telemetria de criação de processos;
- **Agente do Wazuh** — encaminhamento dos eventos do endpoint para o Wazuh.

![Proj-4 - Arquitetura - pipeline do endpoint Windows](/assets/images/proj-4/ARCH-endpoint-investigation 1.png)

### Telemetria de endpoint: tornando a atividade de processos do Windows investigável

Para tornar as execuções de PowerShell analisáveis em um fluxo semelhante ao de um SOC, usei **eventos de criação de processo do Sysmon** no host Windows e os encaminhei ao Wazuh por meio do agente instalado no endpoint.

Isso forneceu os campos mais relevantes para a triagem:

- timestamp;
- host (`agent.name`);
- imagem do processo pai;
- linha de comando;
- linha de comando do processo pai;
- contexto do usuário;
- descrição da regra.

Esses detalhes são o que tornam um alerta de PowerShell realmente útil. Sem eles, o analista fica restrito a um nome genérico de regra e a pouco contexto para tomar uma decisão.

### Sinal gerado em laboratório: quatro execuções seguras de PowerShell

Para tornar o exercício de triagem mais realista, gerei **quatro comandos seguros de PowerShell** no endpoint Windows em um intervalo curto. Todos eram inofensivos, mas um deles foi escolhido de propósito para exigir mais atenção do analista.

Comandos utilizados:

```powershell
powershell.exe -Command "Get-Process | Select-Object -First 5"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
powershell.exe -Command "Get-Service | Select-Object -First 5"
powershell.exe -Command "whoami; hostname; Get-Date"
```

O objetivo não era simular malware. A ideia era produzir um pequeno conjunto de eventos semelhantes e praticar a decisão sobre qual deles justificava uma investigação mais aprofundada.

### Sinal inicial: família de eventos relacionados ao PowerShell no Wazuh

O Wazuh agrupou essas execuções sob a mesma família ampla de detecção relacionada ao PowerShell. Isso é realista: em muitos ambientes de SOC, o primeiro sinal não é uma detecção perfeita, mas um evento amplo que merece análise e ainda depende de contexto adicional.

![Proj-4 - Sinal inicial de PowerShell no Wazuh](/assets/images/proj-4/WIN-ps-initial-alert.png)

<small><em>
Sinal inicial no Wazuh: eventos relacionados ao PowerShell no endpoint Windows que deram início à investigação, mostrando o host, a linha do tempo e a regra correspondente usada como ponto de partida para a triagem.
</em></small>

Nesta etapa, as perguntas do analista eram:

- Qual host foi afetado?
- Qual família de detecção iniciou a revisão?
- Quantos eventos relacionados ocorreram no mesmo intervalo?
- Qual deles merece uma inspeção mais detalhada?

### Revisão das evidências: selecionando a execução mais relevante

Entre os quatro comandos, um se destacou como o principal candidato para uma investigação mais aprofundada:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
```

Esse comando merece mais atenção porque executa o PowerShell com `ExecutionPolicy Bypass` e testa a conectividade com um host e uma porta internos específicos. Embora esse comportamento possa ser legítimo em atividades administrativas, ele também pode aparecer em ações de reconhecimento.

Os outros três comandos também foram capturados pelo Wazuh, mas se pareciam mais com tarefas administrativas rotineiras ou verificações de identidade:

- `Get-Process`;
- `Get-Service`;
- `whoami; hostname; Get-Date`.

É nesse ponto que o projeto deixa de ser apenas “houve uma execução de PowerShell” e se torna um verdadeiro exercício de triagem.

### Evidência bruta do Sysmon: evento de criação de processo

Após selecionar a execução mais relevante, abri no Wazuh o evento bruto de criação de processo registrado pelo Sysmon.

![Proj-4 - Evento bruto do Sysmon - parte 1](/assets/images/proj-4/WIN-ps-raw-event.png)

<small><em>
Evento bruto de criação de processo do Sysmon: timestamp, nome do agente, linha de comando, linha de comando do processo pai, imagem do processo pai, contexto do usuário e descrição da regra do Wazuh.
</em></small>

**Observações de triagem**

- O evento não indicava apenas “atividade de PowerShell”.
- A linha de comando mostrou por que essa execução merecia mais atenção do que as demais.
- O contexto do processo pai ajudou a entender como a execução foi iniciada.
- Essa é a etapa que transforma uma detecção ampla em uma evidência útil para o analista.

### Contexto do processo: atividades próximas no mesmo endpoint

Um único evento bruto é útil, mas não é suficiente por si só. Depois de analisar o evento selecionado, ampliei novamente a visualização no Wazuh para compará-lo com atividades próximas no mesmo host.

![Proj-4 - Contexto do processo no Wazuh](/assets/images/proj-4/WIN-ps-context.png)

<small><em>
Contexto do processo no Wazuh: a execução de PowerShell analisada ao lado de atividades próximas no mesmo host Windows, ajudando a embasar a avaliação do analista.
</em></small>

Essa visão mais ampla foi importante porque mostrou que o evento analisado fazia parte de um pequeno conjunto de atividades próximas no host, em vez de aparecer como uma captura isolada e sem contexto.

**Observações de triagem**

- A regra ampla relacionada ao PowerShell continuava sendo útil, mas a avaliação real dependia dos detalhes do evento e do contexto do host.
- O evento selecionado era o principal candidato para análise devido aos parâmetros usados e à ação orientada à rede.
- As atividades próximas no host não mostraram evidências imediatas de persistência ou de comportamento malicioso subsequente dentro do período analisado.

### Avaliação do analista: encerrar, monitorar, escalonar ou conter?

Este caso foi projetado como um exercício de decisão, não como um exercício de contenção.

Como eu mesma gerei o sinal em um laboratório controlado, já conhecia a causa raiz durante o teste. No entanto, o objetivo era praticar como eu avaliaria o evento se ele aparecesse em uma fila de alertas sem essa certeza prévia.

Meu raciocínio foi:

- o evento **merecia análise**;
- a linha de comando justificava a atenção do analista;
- o contexto do processo estava disponível e podia ser explicado;
- não havia evidências de persistência, acesso a credenciais ou impacto posterior à execução no período analisado;
- o evento, isoladamente, **não justificava contenção**.

Portanto, a disposição mais realista para este caso seria:

- **não encerrar imediatamente sem revisão**;
- **não partir diretamente para contenção**;
- **documentar o contexto identificado**;
- **encerrar como atividade explicada ou manter em monitoramento**, dependendo do ambiente;
- **escalonar somente se surgirem evidências adicionais de atividade suspeita**.

Em um SOC real, essa distinção é importante. Nem todo alerta deve se transformar em um incidente de alta severidade, mas também não deve ser descartado sem análise.

### Próximas melhorias

- Criar uma regra personalizada no Wazuh para padrões mais específicos de PowerShell suspeito ou para relações incomuns entre processos pai e filho.
- Comparar, na mesma linha do tempo do host, execuções claramente benignas com outras que exijam mais atenção.
- Adicionar um segundo caso em que a execução de PowerShell seja seguida por atividade de rede ou persistência.
- Criar uma busca salva ou um dashboard simples para triagem de endpoints Windows.
- Incluir uma seção mostrando como a disposição do alerta mudaria caso o mesmo comportamento aparecesse em vários hosts.
