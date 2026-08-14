---
title: "Triagem de alertas SOC assistida por LLM com Splunk, Sysmon, Python e OpenAI"
date: 2026-07-23
layout: single
lang: pt-BR
translation_key: llm-assisted-soc-triage
author_profile: true
author: julia_pt
sidebar:
  nav: "sidebar_pt"
toc: true
toc_sticky: true
categories: [Laboratórios, SOC, Segurança com IA]
tags:
  [
    Splunk,
    SOC,
    SIEM,
    Sysmon,
    Python,
    OpenAI,
    LLM,
    MITRE-ATT&CK,
    Análise-de-Alertas,
  ]
excerpt: "Um laboratório de triagem SOC com Splunk, Sysmon, Python e OpenAI para estruturar a análise de alertas e comparar a saída do LLM com uma análise manual."
permalink: /pt/laboratorios/triagem-soc-assistida-por-llm/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Fluxo de triagem em que a IA auxilia na análise inicial e os resultados são comparados com uma avaliação manual."
  image_height: 300px
---

### Resumo do projeto

Este projeto testa o uso de um LLM como apoio à triagem inicial de alertas.

O laboratório usa **Windows Event Logs e Sysmon**, enviados ao **Splunk**, com processamento em **Python** e integração com a **API da OpenAI** para gerar avaliações estruturadas e compará-las com uma análise manual baseada nas mesmas evidências.

**Repositório no GitHub:** [AI-ASSISTED-SOC-TRIAGE](https://github.com/Julmgc/AI-ASSISTED-SOC-TRIAGE){: target="\_blank" rel="noopener noreferrer" }

### Ferramentas utilizadas

- **Splunk Enterprise**
- **Sysmon**
- **Windows Event Logs**
- **Python**
- **API da OpenAI**
- **MITRE ATT&CK**

### O que foi desenvolvido

Criei três cenários de alerta envolvendo criação de conta local, execução de PowerShell e um falso positivo controlado.

Cada alerta foi estruturado em JSON e processado por um script em Python, que enviou as evidências disponíveis ao LLM usando um prompt de triagem com restrições definidas.

As avaliações geradas pelo modelo foram então comparadas com uma análise manual baseada nas mesmas evidências.

### Alerta 01: criação de conta local

O caso principal utiliza o **Alerta 01 — criação de nova conta local**.

A atividade gerou o Windows Security Event ID `4720`.

Pesquisei o evento no Splunk:

```spl
index=* source="WinEventLog:Security" EventCode=4720
| table _time host Subject_Account_Name Target_Account_Name
| sort -_time
```

![New local user created event reviewed in Splunk](/assets/images/proj-6/splunk-windows-event-4720.png)

O evento confirmou que a conta `lab_backup` foi criada no host `DESKTOP-HRMT55O` pelo usuário `jules`.

Em seguida, consultei o Sysmon Event ID `1` para obter mais contexto sobre o processo:

```spl
index=* source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
"lab_backup"
| search EventID=1
| table _time host User Image CommandLine ParentImage
```

![Sysmon-Event-ID1](/assets/images/proj-6/Sysmon-Event-ID1,-process-creation.png)

O evento do Sysmon acrescentou a linha de comando e informações sobre o processo pai associadas à criação da conta.

O LLM produziu:

```json
{
  "risk_level": "medium",
  "classification": "needs_review",
  "mitre_attack_mapping": [
    {
      "technique_id": "T1136.001",
      "technique_name": "Create Account: Local Account",
      "confidence": "high"
    }
  ],
  "triage_questions": [
    "Is the user authorized to create local accounts on this endpoint?",
    "Was there an approved change ticket or lab activity?",
    "Was the account later added to a privileged group?",
    "Was the account subsequently used to log in?"
  ],
  "confidence": "medium"
}
```

Minha avaliação manual chegou ao mesmo resultado:

```text
Classificação: needs_review
Nível de risco: médio
```

As evidências confirmaram que a conta havia sido criada, mas não permitiam determinar se a ação era autorizada ou maliciosa.

O LLM também mapeou a atividade para `T1136.001 — Create Account: Local Account` sem concluir, sem evidência adicional, que houve comprometimento ou persistência.

### Casos adicionais

**Alerta 02 — PowerShell suspeito**

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.45 -Port 9997"
```

As duas avaliações classificaram o evento como `needs_review`.

O LLM atribuiu risco `low`, enquanto minha avaliação manual atribuiu `medium`, dando maior peso à combinação de `ExecutionPolicy Bypass` com atividade de conectividade de rede.

O comando, isoladamente, não foi tratado como evidência de comprometimento.

---

**Alerta 03 — falso positivo com alto nível de ruído**

```cmd
cmd.exe /c whoami
```

O comando fazia parte de um teste documentado de visibilidade de execução de comandos no Splunk.

Tanto o LLM quanto a avaliação manual classificaram o evento como `benign`, com risco `low`.

O modelo não tratou o uso de `cmd.exe` ou `whoami`, isoladamente, como evidência de atividade maliciosa.

### Comparação dos resultados

| Alerta                    | IA                             | Avaliação manual               | Resultado             |
| ------------------------- | ------------------------------ | ------------------------------ | --------------------- |
| **01 — Nova conta local** | `needs_review`, risco `medium` | `needs_review`, risco `medium` | Alinhado              |
| **02 — PowerShell**       | `needs_review`, risco `low`    | `needs_review`, risco `medium` | Parcialmente alinhado |
| **03 — Falso positivo**   | `benign`, risco `low`          | `benign`, risco `low`          | Alinhado              |

A principal divergência ocorreu no Alerta 02: a classificação foi a mesma, mas minha avaliação atribuiu maior peso à combinação de `ExecutionPolicy Bypass` com atividade de conectividade de rede.

### Principais conclusões

O LLM foi útil para:

- resumir evidências e destacar observáveis relevantes;
- identificar contexto ausente por meio de perguntas de triagem;
- sugerir mapeamentos MITRE ATT&CK com níveis de confiança;
- manter as conclusões limitadas às evidências disponíveis.

A principal limitação foi a dependência do contexto fornecido. Informações sobre autorização, relações entre processos, resultado da execução e atividade associada poderiam alterar significativamente a avaliação.

O valor do LLM neste laboratório esteve no apoio estruturado à análise, não na tomada autônoma de decisão.

> **IA como apoio à análise, com conclusões limitadas às evidências disponíveis.**
