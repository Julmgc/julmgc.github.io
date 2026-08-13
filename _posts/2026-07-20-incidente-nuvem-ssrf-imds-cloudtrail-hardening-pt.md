---
title: "Investigação de SSRF em AWS: IMDS, CloudTrail e hardening"
date: 2026-07-20-
layout: single
lang: pt-BR
translation_key: cloud-incident-ssrf-imds-cloudtrail-hardening
author_profile: true
author: julia_pt
sidebar:
  nav: "sidebar_pt"
toc: true
toc_sticky: true
categories: [Laboratórios, SOC, Nuvem]
tags:
  [
    AWS,
    EC2,
    CloudTrail,
    SSRF,
    IMDSv2,
    Wazuh,
    OWASP,
    Resposta-a-Incidentes,
    MITRE-ATT&CK,
  ]
excerpt: Um caso em nuvem envolvendo uma tentativa de SSRF contra uma aplicação no EC2, investigação no CloudTrail e mitigação com IMDSv2, bloqueio de destinos internos e princípio do menor privilégio.
permalink: /pt/laboratorios/incidente-nuvem-ssrf-imds-cloudtrail-hardening/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Door Ajar"
  image_height: 300px
---

### Visão geral

Este projeto demonstra um fluxo de investigação de segurança em nuvem usando evidências da aplicação e serviços nativos da AWS:

- implantei uma aplicação Flask em uma instância **EC2** com um endpoint capaz de buscar URLs;
- gerei uma requisição no estilo **SSRF** direcionada ao serviço de metadados da instância;
- revisei atividades de identidade e chamadas de API no **CloudTrail**;
- reduzi o risco exigindo **IMDSv2**, bloqueando destinos internos na aplicação e aplicando **menor privilégio** à função IAM.

![Proj-3 - Arquitetura - investigação de SSRF em nuvem](/assets/images/proj-3/SSRF-PROJ-PORT.png)

### Configuração controlada: aplicação no EC2 e função IAM excessivamente permissiva

Executei um pequeno serviço Flask em uma instância EC2 com um endpoint capaz de buscar URLs. Para fins de laboratório, a instância utilizava uma função IAM com permissões mais amplas do que o necessário, demonstrando como o acesso ao serviço de metadados pode ampliar o impacto de uma falha de SSRF.

Este foi um cenário controlado, utilizando apenas recursos descartáveis e dados não sensíveis.

![Proj-3 - Função IAM anexada à instância EC2](/assets/images/proj-3/EC2-role-attached.png)

### Sinal: requisição no estilo SSRF na telemetria da aplicação

Enviei uma requisição ao endpoint `/fetch` direcionada ao endereço de metadados do EC2 (`169.254.169.254`), simulando uma tentativa de SSRF contra o IMDS.

O foco do projeto foi a investigação, não a exploração. O sinal relevante foi a aplicação ter recebido uma requisição destinada ao serviço de metadados da nuvem.

![Proj-3 - Requisição SSRF registrada em JSON](/assets/images/proj-3/APP-ssrf-log.png)

### Investigação no CloudTrail

O acesso ao endereço `169.254.169.254` era relevante porque o IMDS pode disponibilizar informações da instância e credenciais temporárias associadas à função IAM.

Por isso, revisei no CloudTrail as atividades de identidade e as chamadas de API próximas ao horário da requisição suspeita.

Na janela analisada, identifiquei eventos como:

- `GetCallerIdentity`;
- `DescribeRegions`;
- `DescribeInstances`.

Esses eventos permitiram observar a atividade de identidade e enumeração associada ao período investigado.

![Proj-3 - Sequência no CloudTrail](/assets/images/proj-3/CT-killchain-seq.png)

> Caso as credenciais temporárias tivessem sido expostas, uma chamada como `sts:GetCallerIdentity` seria relevante para confirmar a conta AWS e o contexto da função utilizada.

### Remediação: reduzindo o caminho de risco

Apliquei três medidas de hardening no laboratório para reduzir o caminho de risco entre SSRF e exposição de recursos em nuvem:

- exigi **IMDSv2** na instância EC2;
- bloqueei, no endpoint /fetch, requisições destinadas ao serviço de metadados e a endereços locais;
- reduzi as permissões da função da instância com base no princípio do **menor privilégio**.

![Proj-3 - IMDSv2 obrigatório](/assets/images/proj-3/EC2-imdsv2.png)

Na aplicação, o controle incluía destinos como `169.254.169.254`, `127.0.0.0/8`, `localhost` e faixas `169.254.*`.

Após a alteração, uma nova tentativa de acesso ao serviço de metadados retornou `403 Forbidden`.

![Proj-3 - Controle defensivo contra SSRF](/assets/images/proj-3/APP-egress-or-allowlist.png)

> Como melhoria, esse controle poderia ser ampliado para validar os endereços resultantes da resolução DNS e bloquear outras faixas privadas, loopback, link-local e endereços locais IPv6.

### Conclusão

---

Após o hardening, a mesma requisição passou a retornar `403 Forbidden`, enquanto o sinal continuou visível na telemetria da aplicação. O IMDSv2 também permaneceu obrigatório e a função IAM foi restringida.

O laboratório reuniu um fluxo de investigação em nuvem:

> **detectar → investigar no CloudTrail → remediar → repetir o teste → validar**
