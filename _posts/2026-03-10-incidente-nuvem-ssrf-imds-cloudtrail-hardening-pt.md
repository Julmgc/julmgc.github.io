---
title: "Investigação de incidente em nuvem: sinal de SSRF → risco no IMDS → linha do tempo no CloudTrail → reforço de segurança"
date: 2026-03-10
layout: single
lang: pt-BR
translation_key: cloud-incident-ssrf-imds-cloudtrail-hardening
author_profile: true
author: julia_pt
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
excerpt: "Um caso em nuvem: detectei uma tentativa de SSRF em uma aplicação hospedada no EC2, investiguei a atividade de identidade relacionada no CloudTrail e reduzi o risco com IMDSv2, filtragem de saída e menor privilégio."
permalink: /pt/laboratorios/incidente-nuvem-ssrf-imds-cloudtrail-hardening/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Um fluxo de investigação em nuvem conectando um sinal de SSRF, risco no IMDS, análise no CloudTrail e medidas de hardening."
  image_height: 300px
---

### Visão geral

Este projeto demonstra um fluxo de trabalho focado de SOC em nuvem, usando evidências nativas da AWS:

- implantei uma pequena aplicação Flask em uma instância **EC2**, com um endpoint capaz de buscar URLs (`/fetch?url=...`);
- gerei uma **requisição no estilo SSRF** direcionada ao endereço de metadados do EC2, simulando o risco de exposição de credenciais;
- investiguei atividades relacionadas de **identidade e chamadas de API** no **histórico de eventos do CloudTrail**;
- correlacionei **telemetria da aplicação web** com **evidências da nuvem** de forma compatível com um fluxo de SIEM;
- reduzi o risco exigindo **IMDSv2**, restringindo o **tráfego de saída** e reforçando as permissões de **IAM com menor privilégio**.

![Proj-3 - Arquitetura - investigação de SSRF em nuvem](/assets/images/proj-3/ARCH-ssrf-cloudtrail.png)

<small><em>
Visão simplificada da arquitetura, mostrando a aplicação no EC2, o caminho da requisição no estilo SSRF, o risco relacionado ao serviço de metadados, a investigação no CloudTrail e os controles de hardening.
</em></small>

### Configuração controlada: aplicação no EC2 e contexto de função IAM excessivamente permissiva

Executei um pequeno serviço Flask em uma instância EC2 com um endpoint capaz de buscar URLs. Para fins de laboratório, a instância utilizava uma função IAM com permissões mais amplas do que o necessário, permitindo demonstrar por que o acesso ao serviço de metadados aumentaria o risco caso os controles da aplicação fossem insuficientes.

Este foi um cenário controlado, utilizando apenas recursos descartáveis e dados não sensíveis.

![Proj-3 - Função IAM anexada à instância EC2](/assets/images/proj-3/EC2-role-attached.png)

<small><em>
Contexto do raio de impacto: mostra o instance profile e a função IAM associados à instância EC2, que seriam relevantes caso credenciais temporárias fossem expostas por meio do serviço de metadados.
</em></small>

### Sinal: requisição no estilo SSRF na telemetria da aplicação

Para produzir um sinal útil para um SOC, a aplicação registra cada requisição em formato JSON estruturado. Primeiro, gerei uma requisição normal como linha de base. Em seguida, enviei uma requisição direcionada ao endereço de metadados do EC2 (`169.254.169.254`) para simular uma tentativa de SSRF contra o IMDS.

O foco deste projeto é o fluxo de investigação, não a exploração. O principal sinal de detecção é o fato de a aplicação ter recebido uma requisição suspeita direcionada ao espaço de metadados da nuvem.

![Proj-3 - Requisição SSRF registrada em JSON](/assets/images/proj-3/APP-ssrf-log.png)

<small><em>
Sinal de detecção: telemetria estruturada da aplicação mostrando uma requisição ao endereço de metadados do EC2 (`169.254.169.254`), incluindo a URL solicitada, o IP de origem, o status e o timestamp.
</em></small>

### Por que esse sinal era importante

O endereço IP de metadados é relevante porque o EC2 utiliza o Instance Metadata Service para disponibilizar informações da instância e, quando aplicável, credenciais temporárias associadas à função IAM.

Uma tentativa de alcançar esse endereço por meio de um endpoint de busca de URLs não representa um comportamento normal de usuário e deve ser investigada.

Do ponto de vista do analista, isso levanta perguntas como:

- A requisição foi bloqueada ou permitida?
- A aplicação conseguia alcançar o serviço de metadados?
- Houve alguma chamada suspeita de API da AWS logo depois?
- Qual função IAM poderia ter sido afetada caso credenciais fossem expostas?

### Investigação: linha do tempo no histórico de eventos do CloudTrail

Para construir uma linha do tempo defensável, consultei o histórico de eventos do CloudTrail no período próximo à tentativa de SSRF e revisei as atividades de identidade e de API associadas à função da instância.

Procurei por:

- atividade relacionada ao STS, como `GetCallerIdentity`;
- padrões de enumeração, como chamadas `List*` e `Describe*`;
- acesso ao S3 ou a outros serviços relevantes para a função;
- contexto de sessão e do ator, timestamps e sequência de eventos.

Essa análise permitiu responder às principais perguntas do analista: qual identidade estava ativa, quais ações foram executadas e quão próximas elas ocorreram em relação ao evento web suspeito.

![Proj-3 - Sequência no CloudTrail](/assets/images/proj-3/CT-killchain-seq.png)

<small><em>
Evidência da investigação: o CloudTrail mostra a atividade relevante de identidade e de API alinhada ao período da requisição suspeita.
</em></small>

> Caso a tentativa de SSRF tivesse obtido com sucesso credenciais do serviço de metadados, uma ação comum seria chamar **`sts:GetCallerIdentity`** para confirmar a conta AWS e o contexto da função. Analistas frequentemente procuram essa chamada como um possível indicador inicial de uso indevido de credenciais.

<!-- ## Correlação: primeiro o sinal web, depois a evidência em nuvem

Para manter o projeto fácil de ler por recrutadores, a história de correlação foi mantida intencionalmente simples:

- uma requisição suspeita direcionada a metadados aparece na telemetria da aplicação;
- dentro da mesma janela de investigação, o CloudTrail mostra atividade de identidade e de API relacionada à função da instância EC2.

Isso é suficiente para demonstrar um fluxo de trabalho crível de SOC:
detectar → pivotar → investigar → documentar → reforçar a segurança.

![Proj-3 - Visão de correlação no SIEM](/assets/images/proj-3/WAZUH-correlation.png)

<small><em>
Objetivo da captura: mostrar o fluxo de trabalho do analista em uma visão que conecta a requisição suspeita da aplicação à janela de investigação em nuvem, facilitando triagem e documentação.
</em></small> -->

### Remediação: reduzindo o caminho de risco

Apliquei três medidas concretas de hardening para reduzir o caminho de risco entre SSRF e exposição de recursos em nuvem:

- exigi **IMDSv2** na instância EC2;
- restringi o endpoint de busca para impedir requisições a destinos de metadados e endereços link-local;
- reduzi as permissões da função da instância com base no princípio do **menor privilégio**.

Essas alterações não corrigem apenas um padrão específico de requisição. Elas também reduzem a probabilidade de exploração e o possível raio de impacto de falhas semelhantes.

![Proj-3 - IMDSv2 obrigatório](/assets/images/proj-3/EC2-imdsv2.png)

<small><em>
Controle que interrompe a cadeia de risco: mostra o serviço de metadados do EC2 configurado para exigir IMDSv2.
</em></small>

![Proj-3 - Controle defensivo contra SSRF](/assets/images/proj-3/APP-egress-or-allowlist.png)

<small><em>
Evidência de hardening na aplicação e na rede: mostra que o endpoint de busca foi impedido de alcançar o serviço de metadados e destinos não confiáveis.
</em></small>

### Bloqueio na aplicação para endereços de metadados e redes locais

Em seguida, implementei um controle defensivo simples na aplicação Flask para impedir que o endpoint `/fetch` realizasse requisições a endereços de metadados ou a destinos locais.

O endpoint havia sido criado inicialmente para demonstrar como uma requisição no estilo SSRF poderia alcançar URLs arbitrárias. Para reduzir esse risco, a aplicação passou a bloquear requisições destinadas a:

- `169.254.169.254` — serviço de metadados do EC2;
- `127.0.0.0/8`;
- `localhost`;
- faixas link-local, como `169.254.*`.

Esse controle impede que a aplicação seja utilizada como proxy para acessar endpoints internos sensíveis.

Exemplo da lógica de proteção:

```python
BLOCKED_HOSTS = {
    "169.254.169.254",
    "127.0.0.1",
    "localhost",
}

BLOCKED_PREFIXES = (
    "169.254.",
    "127.",
)
```

Quando um destino bloqueado é solicitado, a aplicação registra o evento e retorna uma resposta `403 Forbidden`.

**<em>Neste projeto, o objetivo foi demonstrar o fluxo de remediação, e não implementar uma defesa de SSRF pronta para produção. Em um ambiente real, eu também validaria os endereços IP resultantes da resolução DNS e restringiria outras faixas privadas, loopback, link-local e endereços locais IPv6, reduzindo o risco de técnicas de bypass dos filtros.</em>**

### Validação: redução do risco antes e depois da correção

Após aplicar o hardening, executei novamente a mesma requisição no estilo SSRF direcionada ao serviço de metadados.

A tentativa continuou visível na telemetria da aplicação, mas já não seguia o mesmo caminho de risco:

- o IMDSv2 era obrigatório na instância;
- a aplicação bloqueava destinos de metadados e endereços locais;
- a requisição retornava `403 Forbidden`, em vez de permitir o acesso ao endpoint de metadados.

Isso confirmou que o padrão suspeito continuava detectável, enquanto o comportamento de risco havia sido reduzido por controles tanto na infraestrutura quanto na aplicação.

### Checklist resumido de investigação

1. **Revisar a telemetria da aplicação:** identificar requisições suspeitas a destinos de metadados ou link-local.
2. **Avaliar o escopo da função:** confirmar qual função IAM ou instance profile estava associado à instância EC2.
3. **Pivotar para o CloudTrail:** revisar atividades próximas de STS e de API para construir a linha do tempo e identificar o ator.
4. **Reduzir a exposição:** exigir IMDSv2, bloquear o acesso ao serviço de metadados pelo endpoint e restringir as permissões da função.
5. **Validar:** executar novamente o teste e confirmar que o caminho de risco foi reduzido após a correção.

**Checklist de triagem em nuvem**

- Revisar os logs da aplicação em busca de requisições direcionadas a metadados.
- Verificar no CloudTrail atividades próximas de STS e chamadas `Describe*` ou `List*`.
- Confirmar que o IMDSv2 é obrigatório.
- Validar que o endpoint de busca bloqueia endereços de metadados e redes locais.
- Registrar a URL suspeita, o IP de origem, os timestamps e as correções aplicadas.

### Próximas melhorias

- Criar uma regra personalizada no Wazuh para sinalizar requisições a endereços de metadados ou faixas link-local.
- Expandir a aplicação para registrar um identificador de requisição, facilitando a correlação entre a aplicação e a linha do tempo da nuvem.
- Criar um pequeno script em Python para resumir eventos do CloudTrail em torno de uma janela de tempo específica.
- Comparar, em um laboratório futuro, mitigações de SSRF no nível do host, da aplicação e da rede.
