---

title: "Github actions: sobre CI e CD"  
slug: CI-CD-Github-actions
date: 2026-07-08  
tags: CI, CD
author: Dev-Lucas10

---

## **GitHub Actions — visão geral sobre CI e CD**

CI (Integração Contínua): automatizar build, testes e análise de código a cada push/PR.
CD (Entrega/Deploy Contínuo): automatizar a publicação do artefato (deploy em staging/produção, criação de imagens Docker, release).

---

## **Como funciona o fluxo básico**

Evento: push, pull_request, release, schedule, workflow_dispatch.

Runner: máquina (hosted ou self-hosted) que executa jobs.

Jobs: unidades paralelizáveis; cada job roda em um runner.

Steps: comandos dentro do job; usam actions (reutilizáveis) ou comandos shell.

Artifacts / Cache / Secrets / Environments: para compartilhar resultados, acelerar builds, proteger credenciais e controlar deploys.

---

## **Boas práticas e recomendações**

Separar CI e CD em workflows distintos para clareza e controle de permissões.

Use secrets para credenciais; nunca commitá-las.

Proteja branches (branch protection rules) e exija checks de CI antes do merge.

Use environments (staging/production) com required reviewers e protection rules para deploys sensíveis.

Cache dependências (actions/setup-node cache, actions/cache) para acelerar builds.

Matrix builds para testar múltiplas versões/combinações.

Fail fast: configure strategy.fail-fast quando apropriado.

Artifacts: armazene binários, relatórios e cobertura para análise posterior.

Logs e observabilidade: envie métricas/erros para Sentry, Datadog, etc., no pipeline.

Least privilege: crie tokens com escopo mínimo; use OIDC para autenticação com cloud providers quando possível (evita secrets estáticos).

Releases automatizadas: use actions/create-release e actions/upload-release-asset para publicar artefatos.

---

## **Estratégias de deploy comuns**

Blue/Green: manter duas versões e alternar tráfego.

Canary: liberar para uma pequena % de usuários e aumentar gradualmente.

Rolling: atualizar instâncias gradualmente.

Immutable infra: criar nova infra (containers/VMs) e substituir a antiga.

---

### **Segurança e compliance**

Use OIDC para autenticar em AWS/GCP/Azure sem armazenar long-lived secrets.

Scan de dependências: dependabot, actions/scan e SCA (Software Composition Analysis).

Secret scanning e token rotation periódica.

Auditoria: habilite logs de auditoria e retenção de artefatos sensíveis conforme política.

---

## **Quando usar self-hosted runners**
Necessário quando precisa de hardware específico (GPU), acesso a rede interna, ou dependências privadas. Garanta atualizações, isolamento e monitoramento.