---
title: "Triagem de alertas SOC assistida por LLM com Splunk, Sysmon, Python e OpenAI"
date: 2026-07-06
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
    Triagem-de-Alertas,
    Engenharia-de-Detecção,
    Resposta-a-Incidentes,
  ]
excerpt: "Um laboratório de triagem SOC com Splunk, Sysmon, Python e OpenAI para estruturar a análise de alertas e comparar a saída do LLM com uma avaliação manual."
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

O laboratório usa **Windows Event Logs e Sysmon**, enviados ao **Splunk**, com processamento em **Python** e integração com a **API da OpenAI** para gerar avaliações estruturadas que podem ser comparadas com uma análise manual.

**Repositório no GitHub:** [AI-ASSISTED-SOC-TRIAGE](https://github.com/Julmgc/AI-ASSISTED-SOC-TRIAGE){: target="\_blank" rel="noopener noreferrer" }

### Ferramentas utilizadas

- **Splunk Enterprise**
- **Sysmon**
- **Windows Event Logs**
- **Python**
- **API da OpenAI**
- **MITRE ATT&CK**

### O que foi desenvolvido

Criei oito cenários de alerta envolvendo PowerShell, falhas de autenticação, atividade web, criação de contas, conexões de saída e falsos positivos.

Cada alerta foi estruturado em JSON e processado por um script em Python, que enviou as evidências disponíveis ao LLM usando um prompt de triagem com restrições definidas.

Este post apresenta um caso em mais detalhes e dois exemplos adicionais. O conjunto completo de alertas, as saídas da IA e as avaliações manuais estão disponíveis no repositório.

### Alerta 05: criação de conta local

O exemplo principal utiliza o **Alerta 05 — criação de nova conta local**.

A atividade gerou o Windows Security Event ID `4720`.

Pesquisei o evento no Splunk:

```spl
index=* source="WinEventLog:Security" EventCode=4720
| table _time host Subject_Account_Name Target_Account_Name
| sort -_time
```

![Evento de criação de uma nova conta local analisado no Splunk](/assets/images/proj-6/splunk-windows-event-4720.png)

O evento confirmou que a conta `lab_backup` foi criada no host `DESKTOP-HRMT55O` pelo usuário `jules`.

Em seguida, consultei o Sysmon Event ID `1` para obter mais contexto sobre o processo:

```spl
index=* source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
"lab_backup"
| search EventID=1
| table _time host User Image CommandLine ParentImage
```

![Sysmon Event ID 1 mostrando o processo responsável pela criação da conta](/assets/images/proj-6/Sysmon-Event-ID1,-process-creation.png)

O evento do Sysmon acrescentou a linha de comando e as informações sobre o processo pai associadas à criação da conta.

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

O LLM também mapeou a atividade para `T1136.001 — Create Account: Local Account` sem fazer conclusões não sustentadas sobre comprometimento ou persistência.

### Casos adicionais

**Alerta 01 — PowerShell suspeito**

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.45 -Port 9997"
```

As duas avaliações classificaram o evento como `needs_review`.

O LLM atribuiu nível de risco `low`, enquanto minha avaliação manual atribuiu `medium`, dando mais peso à combinação de `ExecutionPolicy Bypass` com um comando voltado à conectividade de rede.

---

**Alerta 08 — falso positivo com alto nível de ruído**

```cmd
cmd.exe /c whoami
```

O comando fazia parte de um teste documentado de visibilidade de execução de comandos no Splunk.

Tanto o LLM quanto a avaliação manual classificaram o evento como `benign`, com risco `low`. O modelo não tratou o uso de `cmd.exe` ou `whoami`, isoladamente, como evidência de atividade maliciosa.

### Resultado geral

No conjunto completo de oito alertas, as avaliações da IA e as avaliações manuais ficaram alinhadas em sete casos.

Em um caso houve alinhamento parcial, porque a classificação foi a mesma, mas o nível de risco atribuído foi diferente.

```text
Total de alertas testados: 8
Alinhados: 7
Parcialmente alinhados: 1
```

### Principais conclusões

O LLM foi útil para:

- resumir evidências e extrair observáveis relevantes;
- identificar contexto ausente por meio de perguntas de triagem;
- sugerir mapeamentos MITRE ATT&CK com níveis de confiança;
- estruturar próximos passos de investigação;
- evitar conclusões que não eram sustentadas pelas evidências disponíveis.

A principal limitação foi a dependência do contexto incluído em cada alerta. Informações sobre autorização, árvore de processos, resultado da execução e atividade relacionada eram frequentemente necessárias para chegar a uma conclusão mais forte.

O valor do LLM nesse fluxo, portanto, não está na tomada autônoma de decisão, mas no apoio estruturado à triagem com conclusões limitadas às evidências disponíveis.

> **IA como apoio à análise, com conclusões limitadas às evidências disponíveis.**
