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
excerpt: "Um laboratório prático de triagem SOC usando Splunk, Sysmon, Python e OpenAI para resumir alertas, sugerir mapeamentos MITRE ATT&CK e comparar a saída da IA com a validação manual do analista."
permalink: /pt/laboratorios/triagem-soc-assistida-por-llm/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Fluxo de trabalho de um analista SOC em que a IA auxilia na triagem, enquanto a validação humana permanece como controle final."
  image_height: 300px
---

### Resumo do projeto

Este projeto avalia como um LLM pode auxiliar nas etapas iniciais da triagem de alertas em um SOC.

Construí um pequeno laboratório inspirado em um ambiente de SOC usando **Splunk**, **Sysmon**, **Python** e a **API da OpenAI**. O fluxo coleta telemetria do Windows, extrai o contexto dos alertas, envia evidências estruturadas para um LLM e compara a triagem produzida pela IA com a análise manual do analista.

**Repositório no GitHub:** [AI-ASSISTED-SOC-TRIAGE](https://github.com/Julmgc/AI-ASSISTED-SOC-TRIAGE)

O objetivo não era automatizar decisões finais, mas testar se a IA poderia ajudar o analista a trabalhar com mais rapidez sem chegar a conclusões que não fossem sustentadas pelas evidências.

### Ferramentas utilizadas

- **Splunk Enterprise** — camada de SIEM, busca e investigação
- **Splunk Universal Forwarder** — encaminhamento dos logs do Windows
- **Sysmon** — coleta de telemetria do endpoint
- **Python** — processamento dos alertas e integração com o LLM
- **API da OpenAI** — geração da saída estruturada de triagem
- **MITRE ATT&CK** — mapeamento de técnicas
- **Windows Event Logs** — eventos de autenticação e gerenciamento de contas

### Arquitetura do laboratório

```text
Máquina virtual Windows
  ↓
Sysmon + Windows Event Logs
  ↓
Splunk Universal Forwarder
  ↓
Splunk Enterprise
  ↓
Buscas de triagem em SPL
  ↓
Script em Python
  ↓
Prompt de triagem enviado à OpenAI
  ↓
Avaliação gerada pela IA
  ↓
Validação manual do analista
```

### O que foi desenvolvido

Criei um conjunto de dados com oito alertas inspirados em cenários de SOC:

| Alerta    | Cenário                                   | Objetivo                                                                                     |
| --------- | ----------------------------------------- | -------------------------------------------------------------------------------------------- |
| Alerta 01 | Execução suspeita de PowerShell           | Verificar se o modelo identifica parâmetros suspeitos sem presumir que houve comprometimento |
| Alerta 02 | PowerShell administrativo benigno         | Verificar se o modelo classifica excessivamente uma atividade administrativa normal          |
| Alerta 03 | Sequência de falhas de login              | Avaliar o raciocínio de triagem diante de possível força bruta                               |
| Alerta 04 | Requisição web semelhante a SQL injection | Avaliar a interpretação de um possível ataque a uma aplicação web                            |
| Alerta 05 | Nova conta local criada                   | Avaliar o raciocínio sobre gerenciamento de contas e possível persistência                   |
| Alerta 06 | Conexão de saída suspeita                 | Avaliar a análise conjunta do contexto de processo e rede                                    |
| Alerta 07 | Comando PowerShell codificado             | Avaliar um comportamento de endpoint com risco mais elevado                                  |
| Alerta 08 | Falso positivo com muito ruído            | Verificar se o modelo reconhece quando as evidências são fracas                              |

Cada alerta foi armazenado em um arquivo JSON e processado por um script em Python.

### Validação no Splunk: alerta de criação de conta

Como exemplo principal, utilizei o **Alerta 05: nova conta local criada**.

Esse é um cenário realista para a triagem em SOC porque a criação de contas locais pode fazer parte de uma atividade administrativa legítima, mas também pode indicar persistência quando ocorre sem autorização.

A atividade gerou o evento de segurança do Windows com ID `4720`, que registra a criação de uma conta de usuário.

```text
EventCode: 4720
Tipo de evento: conta de usuário criada
Usuário responsável: DESKTOP-HRMT55O\jules
Usuário criado: lab_backup
Linha de comando: net user lab_backup * /add
```

Primeiro, pesquisei no Windows Security Event Log por eventos de criação de contas:

```spl
index=\* source="WinEventLog:Security" EventCode=4720
| table \_time host EventCode Subject_Account_Name Target_Account_Name Message
| sort -\_time
```

![Evento de criação de uma nova conta local analisado no Splunk](/assets/images/proj-6/splunk-windows-event-4720.png)

A busca confirmou que a atividade de gerenciamento de contas do Windows estava disponível no Splunk e poderia ser analisada em conjunto com o contexto dos processos relacionados.

O evento confirmou que a conta local `lab_backup` foi criada no host `DESKTOP-HRMT55O` pelo usuário `jules`.

No entanto, o Event ID 4720 confirma a ação de gerenciamento da conta, mas não mostra completamente a árvore de processos nem a linha de comando que originou a atividade.

Por isso, também analisei o Sysmon Event ID 1, que registra a criação de processos:

```spl
index=_ source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
"lab_backup"
| rex field=\_raw "<EventID[^>]_>(?<EventID>\d+)</EventID>"
| rex field=\_raw "<Data Name='User'>(?<User>[^<]+)</Data>"
| rex field=\_raw "<Data Name='Image'>(?<Image>[^<]+)</Data>"
| rex field=\_raw "<Data Name='CommandLine'>(?<CommandLine>[^<]+)</Data>"
| rex field=\_raw "<Data Name='ParentImage'>(?<ParentImage>[^<]+)</Data>"
| search EventID=1 CommandLine="_/add_"
| table \_time host EventID User Image CommandLine ParentImage
| sort \_time
```

![Sysmon Event ID 1 com o processo responsável pela criação da conta](/assets/images/proj-6/Sysmon-Event-ID1,-process-creation.png)

### Fluxo de triagem em Python

O script em Python carrega um arquivo JSON de alerta, insere seu contexto em um prompt estruturado, envia os dados para a API da OpenAI e salva a saída produzida pela IA.

Exemplo de execução:

```bash
python main.py --alert alerts/alert_05_new_user_created.json
```

A saída é salva em:

```text
results/ai_outputs/
```

Estrutura do repositório:

```text
ai-assisted-soc-triage/
├── alerts/
├── prompts/
├── results/
│   ├── ai_outputs/
│   └── manual_analysis/
├── main.py
├── requirements.txt
└── .env.example
```

### Exemplo de resultado: nova conta local criada

Para o Alerta 05, o LLM classificou o evento da seguinte forma:

```text
Nível de risco: médio
Classificação: suspeito / requer revisão (needs_review)
Mapeamento MITRE principal: T1136.001 — Create Account: Local Account
```

O resultado foi útil porque o modelo identificou corretamente que a criação de uma conta local pode estar associada à persistência.

No entanto, o modelo não conseguiu determinar se a ação havia sido autorizada. O alerta não incluía um chamado de mudança, uma justificativa de negócio ou uma confirmação de que o usuário estava executando uma atividade administrativa prevista.

Isso tornou o alerta um bom caso de teste: o comportamento era relevante para a segurança, mas as evidências não eram suficientes para confirmar um comprometimento.

### Validação manual do analista

Analisei manualmente cada saída produzida pela IA usando as mesmas evidências.

Para o Alerta 05, minha avaliação foi:

```text
Classificação: requer revisão (needs_review)
Nível de risco: médio
```

Justificativa:

- Uma nova conta local foi criada em um endpoint Windows.
- A criação de contas locais pode estar associada à persistência.
- O comando `net user lab_backup ... /add` é relevante para a segurança e deve ser analisado.
- Não havia evidências de que a conta tivesse sido adicionada ao grupo local de Administradores.
- Não havia evidências de movimento lateral, execução de malware, roubo de credenciais ou exfiltração de dados.
- O alerta não incluía informações de gerenciamento de mudanças, portanto não foi possível confirmar se a ação era autorizada.

A saída da IA estava parcialmente correta. O modelo identificou a preocupação de segurança adequada e sugeriu um mapeamento relevante do MITRE ATT&CK, mas a classificação final ainda dependia da validação do analista.

### Comparação entre a análise humana e a IA

| Alerta                                | Classificação da IA | Classificação manual     | Resultado            |
| ------------------------------------- | ------------------- | ------------------------ | -------------------- |
| PowerShell suspeito                   | Requer revisão      | Requer revisão           | Correto              |
| PowerShell administrativo benigno     | Suspeito            | Benigno / requer revisão | Parcialmente correto |
| Sequência de falhas de login          | Suspeito            | Suspeito                 | Correto              |
| Requisição semelhante a SQL injection | Suspeito            | Suspeito                 | Correto              |
| Nova conta local criada               | Suspeito            | Requer revisão           | Parcialmente correto |
| Conexão de saída suspeita             | Requer revisão      | Requer revisão           | Correto              |
| PowerShell codificado                 | Suspeito            | Suspeito                 | Correto              |
| Falso positivo com muito ruído        | Suspeito            | Benigno / falso positivo | Incorreto            |

Resumo:

```text
Total de alertas testados: 8
Corretos: 5
Parcialmente corretos: 2
Incorretos: 1
```

### Principais conclusões

O LLM foi útil para:

- resumir as evidências do alerta;
- extrair observáveis relevantes;
- sugerir perguntas para orientar a triagem;
- produzir anotações de análise de forma consistente;
- identificar possíveis mapeamentos do MITRE ATT&CK.

O uso do LLM apresentou maior risco quando:

- as evidências eram fracas ou ambíguas;
- ferramentas suspeitas também possuíam usos administrativos legítimos;
- os mapeamentos do MITRE ATT&CK eram plausíveis, mas não estavam totalmente sustentados pelas evidências;
- faltava contexto sobre o ambiente.

### Competências demonstradas pelo projeto

Este laboratório demonstra experiência prática em:

- busca e investigação no Splunk;
- coleta de telemetria do Windows com Sysmon;
- análise do Windows Security Event Log;
- triagem de eventos de gerenciamento de contas;
- extração de contexto de alertas;
- automação com Python;
- integração com a API da OpenAI;
- mapeamento de técnicas do MITRE ATT&CK;
- raciocínio aplicado à triagem de alertas em SOC;
- validação humana de análises de segurança produzidas por IA.

A principal conclusão é que a IA pode auxiliar na primeira etapa da triagem de alertas, mas o analista ainda precisa verificar se as evidências realmente sustentam a conclusão apresentada.

### Conclusão

O projeto demonstrou que um LLM pode acelerar a análise de alertas ao resumir evidências, identificar observáveis e sugerir os próximos passos da investigação.

No entanto, ele não deve ser tratado como responsável pela decisão final.

O papel mais seguro da IA nesse fluxo de trabalho é:

> Auxiliar o analista sem substituir seu julgamento profissional.
