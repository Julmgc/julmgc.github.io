---
title: "Mini laboratório SOC: triagem web, detecção e contenção com Wazuh"
date: 2026-02-16
layout: single
lang: pt-BR
translation_key: mini-soc-wazuh-hardening
author_profile: true
author: julia_pt
toc: true
toc_sticky: true
categories: [Laboratórios, SOC]
tags: [Wazuh, SIEM, pfSense, Proxmox, Engenharia-de-Detecção]
excerpt: "Um miniambiente corporativo em Proxmox: rede segmentada, telemetria para SOC com Wazuh e syslog do pfSense, detecções personalizadas e hardening."
permalink: /pt/laboratorios/mini-soc-wazuh-hardening/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Um fluxo completo de SOC, desde a geração do sinal até a detecção, triagem, contenção e validação."
  image_height: 300px
---

## Visão geral

Este projeto apresenta um mini laboratório SOC baseado em Proxmox e demonstra um fluxo completo no Wazuh:

> **sinal → detecção → triagem → contenção → validação**

Neste laboratório, eu:

- gerei sinais realistas de ataques web e de autenticação em uma rede segmentada;
- investiguei esses sinais no Wazuh usando telemetria estruturada da aplicação;
- escalonei um alerta personalizado de correlação para SQL injection (`rule.id:100200`);
- apliquei contenção no host e validei o resultado por meio de uma redução clara nos alertas antes e depois da mudança.

**Evidências produzidas:** consultas no Discover, detalhes dos alertas, dashboard de SQL injection, captura com `tcpdump` do tráfego lateral e regras do UFW.

## Por que este cenário é realista para um SOC

- Os alertas são sustentados por **telemetria estruturada da aplicação web**, com logs JSON por requisição, e não apenas por capturas de ferramentas.
- As detecções foram validadas por meio de consultas no Wazuh Discover, detalhes dos alertas e verificações pontuais, como `tcpdump` e estado do firewall.
- A contenção considerou o **caminho real do tráfego de rede**, especialmente o tráfego lateral, e foi validada com a repetição do teste e a redução dos alertas.

## Arquitetura do laboratório

Implantei **quatro máquinas virtuais** para este projeto:

- **Wazuh (SIEM)** — manager, indexer e dashboard em uma instalação all-in-one;
- **UbuntuApp (servidor web)** — Nginx, aplicação Flask e agente do Wazuh, produzindo telemetria web e de autenticação;
- **pfSense (firewall/gateway)** — segmentação, políticas de firewall e fonte de syslog;
- **Jumpbox (bastion e gerador de tráfego)** — origem de todo o tráfego de teste na VLAN20 (`192.168.20.10`).

![Baseline A - Layout do laboratório no Proxmox](/assets/images/proj-1/PROXMOX-lab-layout.png)

## Topologia de rede relevante para o estudo de caso

Todo o tráfego apresentado foi gerado dentro da VLAN20. O endereço IP de origem observado pelo servidor web corresponde ao host que realmente enviou as requisições HTTP.

- **VLAN20 (`192.168.20.0/24`)** — rede interna do laboratório em que o tráfego foi gerado e observado;
  - **Jumpbox — origem das requisições e simulador de atacante:** `192.168.20.10`;
  - **UbuntuApp — servidor web alvo:** `192.168.20.50`;
  - **pfSense — gateway e firewall:** `192.168.20.1`;
  - **Wazuh — SIEM:** alertas indexados em `wazuh-alerts-*`.

> **Observação de SOC:** em segmentos L2 planos, o tráfego lateral pode não atravessar o firewall. Por esse motivo, a **telemetria do endpoint e da aplicação**, como agente do Wazuh, logs de autenticação e logs da aplicação, torna-se a principal fonte de evidências para investigação e decisões de contenção.

## Telemetria: tornando a aplicação adequada para um SOC

**Nginx como proxy reverso**

A UbuntuApp executa uma aplicação Flask atrás do **Nginx**, utilizado como proxy reverso. O Nginx encerra a conexão do cliente e encaminha a requisição para o serviço da aplicação.

**Logs estruturados da aplicação Flask**

Para produzir sinais úteis para o SOC, implementei logs estruturados em JSON dentro da aplicação Flask:

- um evento JSON por requisição, facilitando análise e busca;
- campos relevantes para o SOC, como `srcip`, `method`, `path`, `query_string` truncada, `status`, `duration_ms` e `user_agent`;
- rotação de logs para impedir crescimento ilimitado;
- registro correto do IP do cliente atrás do Nginx, considerando um único salto de proxy confiável.

**Localização do log na UbuntuApp:**

```text
/var/log/flaskapp/app.log
```

![Baseline A - Eventos JSON da aplicação Flask](/assets/images/proj-1/FLASK-JSON-LOG-EVENTS.png)

<small><em>
Evidência de telemetria bruta: logs JSON de requisições gravados pela aplicação Flask em <code>/var/log/flaskapp/app.log</code>. O exemplo mostra requisições ao endpoint <code>/search</code> filtradas para validar os campos <code>srcip</code>, <code>path</code>, <code>status</code> e <code>query_string</code> utilizados nas detecções do Wazuh.
</em></small>

## Engenharia de detecção com regras personalizadas do Wazuh

Criei e testei regras personalizadas no arquivo `local_rules.xml` para demonstrar um trabalho prático de engenharia de detecção, indo além da simples instalação das ferramentas.

<a href="https://github.com/Julmgc/detections/blob/main/wazuh/local_rules.xml" target="_blank" rel="noopener noreferrer">
🔗 Wazuh `local_rules.xml`
</a>

**Sinal de força bruta em autenticação: falhas no endpoint `/login`**

Para simular um sinal realista de ataque contra autenticação, gerei aproximadamente 20 tentativas de login malsucedidas a partir da Jumpbox (`192.168.20.10`) contra a UbuntuApp (`192.168.20.50`).

Isso produziu um pico claro de respostas HTTP `401` e criou uma trilha pesquisável no SIEM.

**Comando JSON POST — 20 tentativas com intervalo de 3 segundos:**

```bash
TARGET="http://UbuntuApp_IP"
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Content-Type: application/json" \
    -X POST "$TARGET/login" \
    --data '{"username":"demo","password":"wrongpass"}'
  sleep 3
done
```

Resultado esperado: respostas `401` repetidas.

![Baseline A - Tentativas de login](/assets/images/proj-1/LOGIN-ATTEMPTS.png)

**Wazuh Discover — DQL para logs da UbuntuApp, com origem na Jumpbox:**

Índice:

```text
wazuh-alerts-*
```

Consulta:

```text
agent.name:"ubuntu-app" and data.path:"/login" and data.status:401 and data.srcip:"192.168.20.10" and data.app:"ticketing-lab" and data.method:"POST"
```

![Baseline A - Contagem de eventos no Wazuh](/assets/images/proj-1/WAZUH_COUNT_UP.png)

**Observações de triagem**

- O padrão corresponde a uma rajada de tentativas semelhante a credential stuffing: várias respostas `401` provenientes da mesma origem em um curto intervalo.
- Os campos principais foram `agent.name`, `data.srcip`, `data.path`, `data.status`, `data.app` e `data.method`.
- Em produção, o próximo ajuste seria gerar um alerta apenas após `N` falhas em `M` minutos por `data.srcip` e excluir scanners internos conhecidos.

## Telemetria de reconhecimento: caminhos sensíveis comuns

Para simular reconhecimento web automatizado, gerei uma pequena rajada de requisições a caminhos frequentemente procurados durante varreduras, como:

- `/wp-login.php`;
- `/.git/config`;
- `/.env`;
- `/admin`;
- `/phpmyadmin`;
- `/server-status`.

As requisições foram enviadas da Jumpbox para a UbuntuApp.

![Baseline A - Rajada rápida de requisições](/assets/images/proj-1/QUICK BURST.png)

No Wazuh, isso produziu um sinal claro de reconhecimento, que pôde ser analisado rapidamente por meio dos campos `data.srcip` e `data.path`.

**Consulta DQL no Wazuh Discover:**

```text
agent.name:"ubuntu-app" and
data.app:"ticketing-lab"
and data.event:"http_request"
and data.srcip:"192.168.20.10"
and (data.path:"/wp-login.php" or data.path:"/admin" or data.path:"/.git/config" or data.path:"/phpmyadmin" or data.path:"/.env" or data.path:"/server-status")
```

![Baseline A - Eventos encontrados no Wazuh Discover](/assets/images/proj-1/wazuh_discover_hits.png)

**Observações de triagem**

- Uma rajada de requisições para vários caminhos sensíveis em um curto intervalo é típica de reconhecimento.
- Respostas `404` isoladas são ruído comum; a diversidade de caminhos é um sinal mais relevante.
- O próximo ajuste seria alertar com base na combinação entre diversidade de caminhos e taxa de requisições, mantendo uma allowlist para scanners confiáveis.

## Sinal semelhante a SQL injection no endpoint `/search`

Para gerar um sinal realista de ataque web, com foco em detecção e não em exploração, enviei um pequeno conjunto de requisições semelhantes a SQL injection para o endpoint interno `/search`, a partir da Jumpbox.

![Baseline A - Payloads](/assets/images/proj-1/payloads.png)

No Wazuh Discover, filtrei os logs HTTP da UbuntuApp para confirmar as requisições ao endpoint `/search` e capturar exatamente a query string utilizada.

**Consulta DQL:**

```text
agent.name:"ubuntu-app"
data.app:"ticketing-lab"
and data.path:"/search"
and data.srcip:"192.168.20.10"
and data.query_string:"q=%27%20OR%201%3D1%20--"
```

![Baseline A - SQL injection no Wazuh](/assets/images/proj-1/wazuh-SQLi.png)

## Engenharia de detecção: regra personalizada de alerta

O Wazuh já possui uma regra nativa para padrões web relacionados a SQL injection: `31164`, de nível 6.

Para tornar o sinal mais útil operacionalmente para um SOC, criei uma regra local de correlação:

- **ID da regra:** `100200`;
- **nível:** `12`;
- **condição:** acionada quando a regra `31164` dispara;
- **mapeamento MITRE ATT&CK:** `T1190 — Exploit Public-Facing Application`.

A regra aumenta a severidade e facilita a priorização do alerta.

![Baseline A - Regra 100200 no local_rules.xml](/assets/images/proj-1/RULE.ID-100200-local_rules.png)

Abaixo, o alerta correlacionado disparando no Wazuh Discover:

![Baseline A - Regra 100200 no Wazuh](/assets/images/proj-1/ruleID100200.png)

**Observações de triagem**

- A correlação `31164 → 100200` reduz ruído e melhora o roteamento, transformando um indicador de menor confiança em um alerta de prioridade mais alta.
- O próximo ajuste seria disparar a regra `100200` somente após pelo menos três eventos em dois minutos por `srcip`, reduzindo alertas causados por tentativas isoladas.

## Dashboard de SOC: detecção de SQL injection

Para tornar a investigação repetível e semelhante a uma operação de SOC, criei um pequeno dashboard no Wazuh focado nos alertas de SQL injection.

O dashboard destaca:

- quando ocorreu o pico;
- qual origem gerou os eventos, usando `data.srcip`;
- qual payload ou URL acionou a detecção, usando `data.url`.

![Baseline A - Dashboard SOC de ataques web](/assets/images/proj-1/E4-SOC-WEB-ATTACKS.png)

## Ferramenta auxiliar do analista: resumo rápido dos alertas

Para acelerar a primeira etapa da triagem, utilizei um pequeno script em Python que resume os logs JSON da aplicação Flask por:

- principais endereços IP de origem;
- caminhos mais acessados;
- códigos de status;
- query strings.

Isso facilitou a confirmação de que uma única origem dominava a janela analisada e de que a atividade incluía falhas repetidas de login, reconhecimento de caminhos sensíveis e requisições semelhantes a SQL injection para `/search`.

🔗 Script auxiliar `summarize_flask_logs.py`: [repositório no GitHub](https://github.com/Julmgc/detections/blob/main/helpers/summarize_flask_logs.py)

Exemplo de uso:

```bash
python3 summarize_flask_logs.py /var/log/flaskapp/app.log
```

![Baseline A - Resumo de triagem em Python](/assets/images/proj-1/python-triage-summary.png)

<small><em>
Saída da ferramenta auxiliar: resumo dos principais IPs de origem, caminhos, códigos de status e query strings observados durante a janela de investigação.
</em></small>

## Hardening e validação: contenção da Jumpbox

Embora tenham sido criadas regras no pfSense (`192.168.20.1`) para restringir o tráfego da Jumpbox, o fluxo HTTP **não atravessava o firewall**.

A captura com `tcpdump` na UbuntuApp mostrou pacotes `TCP/80` partindo de `192.168.20.10` — Jumpbox — para `192.168.20.50` — UbuntuApp. Isso confirmou que o tráfego era comutado em camada 2 dentro da VLAN20.

![Baseline A - Tráfego entre Jumpbox e UbuntuApp](/assets/images/proj-1/jumpbox-ubuntu-curl-traffic.png)

**Controle aplicado no endpoint**

Configurei o UFW na UbuntuApp para bloquear conexões `TCP/80` provenientes de `192.168.20.10`, mantendo o SSH disponível para gerenciamento.

![Baseline A - Status do UFW](/assets/images/proj-1/ufw-status.png)

**Validação**

Após aplicar o controle, repeti a mesma requisição. O resultado foi uma redução clara nos alertas de SQL injection associados à regra `100200`.

![Baseline A - Dashboard do Wazuh após contenção](/assets/images/proj-1/wazuh-dashboard-SQLi.png)

**Cadeia de evidências validada**

- O alerta foi disparado por `rule.id:100200`.
- A requisição foi confirmada na telemetria por meio de `data.url` e `full_log`.
- O caminho real do tráfego lateral foi confirmado com `tcpdump`.
- O controle foi implementado no host com uma regra UFW bloqueando `TCP/80` da origem `192.168.20.10`.
- O resultado foi validado com a repetição do teste, o bloqueio da requisição e a redução dos alertas.

## Mini playbook de SOC

1. **Definir o escopo:**

   ```text
   agent.name:"ubuntu-app" and rule.id:"100200"
   ```

2. **Identificar origem e payload:** verificar `data.srcip` e `data.url`.
3. **Conter:** bloquear `TCP/80` de `192.168.20.10` na UbuntuApp usando UFW.
4. **Validar:** repetir a requisição e confirmar a redução posterior nos alertas de `rule.id:100200`.

## Próximas melhorias

- Adicionar correlação por limiar, disparando a regra `100200` somente após pelo menos três indicadores de SQL injection em dois minutos por `srcip`.
- Adicionar rastreamento de requisições com um `request_id` preservado do Nginx até os logs da aplicação.
- Separar Jumpbox e UbuntuApp em VLANs diferentes para comparar contenção no host com contenção na rede e garantir que o tráfego atravesse o gateway.
