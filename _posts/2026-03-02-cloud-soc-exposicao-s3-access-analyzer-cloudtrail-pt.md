---
title: "Cloud SOC MVP: detecção, alerta e investigação de exposição pública no S3 com Access Analyzer + CloudTrail"
date: 2026-03-02
layout: single
lang: pt-BR
translation_key: cloud-soc-s3-exposure-access-analyzer-cloudtrail
author_profile: true
author: julia_pt
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
excerpt: "Um caso em nuvem: expus intencionalmente um bucket S3, detectei a exposição com o IAM Access Analyzer, disparei um alerta com EventBridge + SNS, investiguei a alteração no CloudTrail e corrigi o problema restaurando o Block Public Access e removendo a política pública do bucket."
permalink: /pt/laboratorios/cloud-soc-exposicao-s3-access-analyzer-cloudtrail/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Um fluxo de SOC em nuvem conectando exposição pública no S3, detecção com Access Analyzer, alerta com EventBridge e SNS, investigação no CloudTrail e remediação."
  image_height: 300px
---

## Visão geral

Este projeto demonstra um fluxo de investigação em nuvem usando evidências nativas da AWS:

- criei, de forma controlada, uma **exposição pública em um bucket S3** usando um bucket descartável e um arquivo sem dados sensíveis;
- detectei o problema com o **IAM Access Analyzer**;
- configurei **EventBridge + SNS** para enviar um alerta por e-mail quando ocorressem alterações relacionadas a acesso público no S3;
- investiguei a mudança no **histórico de eventos do CloudTrail** para confirmar **o que foi alterado, quando e por qual identidade**;
- corrigi a exposição restaurando o **Block Public Access** e removendo a política pública do bucket;
- validei que o bucket não estava mais acessível publicamente.

![Proj-2A - Arquitetura - pipeline AWS para SOC](/assets/images/proj-2/ARCH-aws-to-soc.png)

<small><em>
Visão do pipeline mostrando: exposição no S3 → detecção pelo Access Analyzer → alerta com EventBridge + SNS → investigação no CloudTrail → remediação.
</em></small>

## Por que isso importa para um analista de SOC

A exposição pública de buckets S3 é um problema comum de segurança em nuvem porque pode tornar dados armazenados acessíveis pela internet de forma não intencional.

Do ponto de vista do analista, as principais tarefas são:

- confirmar a exposição;
- identificar qual controle permitiu o acesso público;
- determinar quem realizou a alteração;
- garantir que a equipe seja notificada a tempo;
- verificar se o estado de risco foi completamente removido.

Este projeto se concentra exatamente nesse fluxo: detecção, alerta, construção da linha do tempo, atribuição, remediação e validação.

## Configuração incorreta controlada: leitura pública de um objeto de teste

Criei um bucket descartável e enviei um único arquivo de teste (`dummy.txt`). Em seguida, relaxei intencionalmente o **Block Public Access** e apliquei uma política de bucket concedendo `s3:GetObject` a todos os principals no caminho do objeto de teste.

![Proj-2A - Permissões do S3 mostrando exposição pública](/assets/images/proj-2/S3-public-permissions.png)

<small><em>
O bucket se tornou publicamente legível porque o Block Public Access foi desativado e a política do bucket concedeu `s3:GetObject` a todos os principals.
</em></small>

> **Observação de SOC:** nenhum conteúdo sensível foi armazenado no bucket; a exposição existiu apenas por um curto período durante o laboratório.

## Detecção: finding do IAM Access Analyzer

O IAM Access Analyzer produziu o principal sinal de detecção deste caso. Ele identificou que o bucket estava acessível publicamente, relacionou a exposição à política do bucket e mostrou que todos os principals possuíam acesso de leitura por meio de `s3:GetObject`.

![Proj-2A - Finding do Access Analyzer para exposição pública no S3](/assets/images/proj-2/AA-public-bucket-finding.png)

<small><em>
Sinal de detecção: o IAM Access Analyzer marcou o bucket S3 como publicamente acessível, mostrando acesso de leitura público e ativo concedido a todos os principals por meio da política do bucket.
</em></small>

## Investigação: histórico de eventos do CloudTrail

Para construir uma linha do tempo defensável do incidente, revisei o **histórico de eventos do CloudTrail** no período próximo à exposição e filtrei por alterações de configuração do S3 que afetavam o bucket.

Procurei por:

- alterações na política do bucket (`PutBucketPolicy` e `DeleteBucketPolicy`);
- alterações no bloqueio de acesso público (`PutPublicAccessBlock` e `DeletePublicAccessBlock`);
- contexto de identidade, para identificar quem executou a ação;
- evidências que mostrassem exatamente o que mudou e em qual horário.

![Proj-2A - Evidência no CloudTrail de alteração de política/acesso público no S3](/assets/images/proj-2/CT-s3-policy-change.png)

<small><em>
Evidência da investigação: o CloudTrail registra o evento `PutBucketPolicy` com timestamp, ator e bucket S3 afetado, fornecendo uma linha do tempo defensável para a exposição.
</em></small>

## Alerta: notificação por e-mail com EventBridge + SNS

Para tornar o fluxo mais próximo de uma operação real, adicionei uma etapa de alerta para mudanças relacionadas a acesso público no S3.

Neste laboratório, o fluxo de alerta foi:

- uma **trail ativa do CloudTrail** registrou eventos de gerenciamento do S3;
- o **EventBridge** identificou alterações de configuração relacionadas à exposição pública;
- o **SNS** enviou um e-mail com o bucket afetado, a ação de API e o horário do evento.

Isso adicionou uma camada operacional de monitoramento ao projeto: a mudança de risco não ficou apenas disponível para consulta nas ferramentas da AWS, mas também gerou uma notificação ativa quando ocorreu.

![Proj-2A - Regra do EventBridge para alertas de mudanças no S3](/assets/images/proj-2/EVB-s3-public-alert-rule.png)

<small><em>
Configuração do alerta: regra do EventBridge preparada para identificar eventos de gerenciamento do CloudTrail relacionados a mudanças na política do S3 e no bloqueio de acesso público.
</em></small>

![Proj-2A - Alerta por e-mail do SNS para mudança de acesso público no S3](/assets/images/proj-2/SNS-s3-alert-email.png)

<small><em>
Evidência operacional do alerta: e-mail do SNS disparado pelo fluxo CloudTrail → EventBridge após uma alteração `PutBucketPolicy`, mostrando o serviço AWS, a ação de API e o horário do evento.
</em></small>

## Remediação: revertendo a exposição pública

Correções aplicadas:

- reativei o **Block Public Access**;
- removi a política pública do bucket.

## Validação: confirmando que o bucket deixou de ser público

Após a remediação, verifiquei que:

- o **Block Public Access** havia sido restaurado;
- a política pública do bucket havia sido removida;
- o bucket não permitia mais leitura pública;
- o estado de configuração insegura já não estava presente.

![Proj-2A - S3 corrigido, sem acesso público](/assets/images/proj-2/S3-remediated.png)

<small><em>
Encerramento: após a remediação, o Block Public Access foi restaurado e a política pública foi removida, impedindo novamente a leitura pública do bucket.
</em></small>

## Checklist resumido de investigação em nuvem

1. **Confirmar a exposição** no S3 e no Access Analyzer.
2. **Revisar o alerta** recebido por EventBridge + SNS.
3. **Construir a linha do tempo** no CloudTrail.
4. **Atribuir a alteração** à identidade responsável e ao timestamp correspondente.
5. **Remediar** restaurando o Block Public Access e removendo a política pública.
6. **Validar** que o bucket não está mais acessível publicamente.
