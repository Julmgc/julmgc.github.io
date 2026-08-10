---
title: "Exposição pública no S3 com Access Analyzer e CloudTrail"
date: 2026-03-02
layout: single
lang: pt-BR
translation_key: cloud-soc-s3-exposure-access-analyzer-cloudtrail
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
    S3,
    CloudTrail,
    EventBridge,
    SNS,
    IAM-Access-Analyzer,
    Resposta-a-Incidentes,
    Segurança-em-Nuvem,
  ]
excerpt: "Laboratório de segurança em nuvem para detectar, investigar e corrigir uma exposição pública no S3 usando Access Analyzer, CloudTrail, EventBridge e SNS."
permalink: /pt/laboratorios/cloud-soc-exposicao-s3-access-analyzer-cloudtrail/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Fluxo de detecção, investigação, alerta e remediação de uma exposição pública no Amazon S3."
  image_height: 300px
---

### Resumo do projeto

Neste laboratório, criei uma exposição pública controlada em um bucket S3 e usei serviços nativos da AWS para detectar a configuração insegura, investigar a alteração, gerar um alerta e restaurar o estado seguro do bucket.

O **IAM Access Analyzer** identificou a exposição, o **CloudTrail** forneceu o histórico da alteração e **EventBridge + SNS** geraram uma notificação por e-mail.

![Arquitetura do fluxo de segurança na AWS](/assets/images/proj-2/RESUMO-DO-PROJ.png)

### Exposição pública controlada

Criei um bucket descartável contendo apenas um arquivo de teste (`dummy.txt`).

Em seguida, desativei temporariamente as proteções relevantes do **Block Public Access** e apliquei uma política de bucket permitindo `s3:GetObject` publicamente para o objeto de teste.

![Permissões do S3 mostrando a exposição pública](/assets/images/proj-2/S3-public-permissions.png)

Nenhum dado sensível foi armazenado no bucket durante o laboratório.

### Detecção com IAM Access Analyzer

O **IAM Access Analyzer** identificou que o bucket estava acessível publicamente.

O finding mostrou que a exposição era causada pela política do bucket e que havia acesso público de leitura por meio de `s3:GetObject`.

![Finding do IAM Access Analyzer mostrando exposição pública no S3](/assets/images/proj-2/AA-public-bucket-finding.png)

O finding confirmou que o bucket estava exposto publicamente por meio da política configurada.

### Investigação com CloudTrail

Depois da detecção, revisei o **CloudTrail** para confirmar a alteração que havia causado a exposição pública.

Como a configuração insegura havia sido criada pela política do bucket, localizei o evento `PutBucketPolicy` e revisei o timestamp, a identidade responsável e o recurso afetado.

![Evidência no CloudTrail de alteração da política do S3](/assets/images/proj-2/CT-s3-policy-change.png)

O evento confirmou a alteração da política e permitiu associá-la à identidade que executou a chamada.

### Alerta com EventBridge + SNS

Além da detecção no Access Analyzer, configurei um fluxo de alerta para alterações relacionadas ao acesso público no S3.

Uma trail do **CloudTrail** registrou os eventos de gerenciamento, o **EventBridge** filtrou alterações relevantes na configuração do S3 e o **SNS** enviou uma notificação por e-mail.

![Regra do EventBridge para alterações de acesso no S3](/assets/images/proj-2/EVB-s3-public-alert-rule.png)

<small><em>
Regra do EventBridge configurada para capturar eventos do CloudTrail relacionados a alterações na política do bucket e no Block Public Access.
</em></small>

![Notificação por e-mail enviada pelo SNS](/assets/images/proj-2/SNS-s3-alert-email.png)

<small><em>
Notificação do SNS gerada após um evento `PutBucketPolicy`, incluindo a ação de API, horário, identidade e detalhes do bucket afetado.
</em></small>

A notificação apresentou a ação de API, o horário do evento e o contexto da alteração registrada.

### Remediação e validação

Para corrigir a exposição:

- reativei o **Block Public Access**;
- removi a política pública do bucket.

Depois da remediação, validei que o bucket não permitia mais leitura pública e que a configuração insegura havia sido removida.

![Bucket S3 após a remediação](/assets/images/proj-2/S3-remediated.png)

<small><em>
Validação após a remediação: o Block Public Access está ativado e a política pública do bucket foi removida.
</em></small>

### Conclusão

O laboratório integrou serviços nativos da AWS para detectar, investigar e corrigir uma exposição pública no S3.

O **IAM Access Analyzer** identificou a exposição, o **CloudTrail** forneceu o histórico da alteração e da identidade responsável, e **EventBridge + SNS** transformaram a mudança de configuração em uma notificação.

Após a correção, validei que o **Block Public Access** estava ativo novamente e que a política pública havia sido removida.
