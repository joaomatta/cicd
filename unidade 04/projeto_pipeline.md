# Estudo de Caso DevOps

## Contexto Geral

Este estudo de caso descreve a concepção e implementação de um pipeline robusto de CI/CD utilizando GitHub Actions em um cenário corporativo realista. O objetivo é servir como material didático avançado para análise, discussão e avaliação de práticas DevOps, indo além de exemplos simplificados e abordando decisões arquiteturais, riscos, trade-offs e governança.

O cenário proposto envolve uma empresa de médio porte que opera uma plataforma digital orientada a APIs, com crescimento acelerado, múltiplos times de desenvolvimento e pressão crescente por qualidade, segurança e previsibilidade de entregas.

## Especificação do Caso

A aplicação em questão é uma API backend responsável por operações de cadastro, consulta e processamento de dados de clientes corporativos. Trata-se de um serviço crítico para o negócio, consumido por aplicações web, mobile e integrações B2B.

A stack tecnológica adotada é composta por Node.js (versão LTS), framework Express, testes automatizados com Jest, linting com ESLint, análise estática de segurança via CodeQL, containerização com Docker e publicação de imagens em um registry privado. O deploy é realizado em um cluster Kubernetes gerenciado, com ambientes separados para desenvolvimento, homologação e produção.

O repositório GitHub segue uma organização padrão, contendo diretórios bem definidos para código-fonte, testes, Dockerfile, scripts auxiliares e configuração de workflows. A estratégia de versionamento adota branches de feature, uma branch principal de integração e branches de release, exigindo controle rigoroso de qualidade antes de qualquer promoção entre ambientes.

Entre os requisitos não funcionais destacam-se a necessidade de rastreabilidade completa entre commits, builds e deploys, aplicação consistente de políticas de segurança, execução obrigatória de testes automatizados, controle de acesso a ambientes sensíveis e feedback rápido para os desenvolvedores.

O principal problema que motivou a evolução do pipeline foi a existência de processos manuais, inconsistentes entre times, com falhas recorrentes em produção, ausência de gates de qualidade e dependência excessiva de conhecimento tácito de pessoas-chave.

## Objetivos do Pipeline

O pipeline de CI/CD tem como objetivo central padronizar e automatizar o ciclo de vida de entrega da aplicação, reduzindo riscos operacionais e aumentando a confiabilidade das releases. Ele deve garantir que todo código integrado passe por validações automáticas de qualidade, segurança e conformidade antes de ser promovido para ambientes superiores.

Além disso, o pipeline precisa oferecer isolamento claro entre ambientes, permitir deploys controlados com aprovação explícita em produção, reutilização de componentes de automação, observabilidade do processo e facilidade de evolução futura.

## Questões Instrumentais e Evolutivas (Mão na Massa)

As questões a seguir são progressivas e interdependentes. Cada uma exige que o aluno estenda os mesmos arquivos YAML, mantendo coerência com decisões anteriores. O resultado esperado é um pipeline robusto, legível e governável. Em cada questão há links de referência no portal de documentação do GitHub Actions.

Questão 1 – Workflow mínimo de Lint como primeiro gate

Implemente um workflow de CI acionado em pull requests para a branch principal, contendo um job exclusivo para lint. O job deve fazer checkout do repositório, configurar o Node.js (LTS), instalar dependências com reprodutibilidade e executar o lint.

Referências:
* https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax

* https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow

Artefato esperado: criação de .github/workflows/ci.yml com o job de lint.

Questão 2 – Adição de testes automatizados em job separado e dependência explícita

Evolua o workflow criando um job separado para testes automatizados. O job de testes deve depender do sucesso do lint, de forma explícita, usando dependências entre jobs. O aluno deve justificar por que o lint é colocado antes dos testes neste caso.

Referências:
* https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-jobs

Artefato esperado: atualização do mesmo .github/workflows/ci.yml com jobs.<job_id>.needs.

Questão 3 – Otimização com cache de dependências e redução do tempo de feedback

Evolua o workflow para reduzir o tempo total de execução. Aplique cache de dependências do Node, de forma compatível com npm ci, mantendo consistência entre jobs. Explique o que pode dar errado se o cache for configurado de forma inadequada.

Referências:
* https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching 
* https://docs.github.com/en/actions/concepts/workflows-and-actions/dependency-caching

Artefato esperado: ci.yml com cache bem definido e explicado no próprio YAML por comentários.

Questão 4 – Build de imagem Docker como artefato imutável após gates de qualidade

Adicione um job de build de imagem Docker que só execute após lint e testes passarem. A imagem deve ser versionada com tag imutável baseada no SHA do commit. O aluno deve explicar como essa decisão habilita rastreabilidade e promoção do mesmo artefato nos deploys.

Referências:
* https://docs.github.com/en/actions/concepts/workflows-and-actions/workflows (para visão geral de jobs) 
* https://docs.github.com/en/actions/reference/workflows-and-actions/contexts (para uso de github.sha)

Artefato esperado: ci.yml com job de build, dependências corretas e comentários.

Questão 5 – Gestão de segredos e princípio do menor privilégio

Introduza no pipeline a necessidade de autenticação em um registry privado (ou no GitHub Container Registry). O aluno deve implementar login seguro e explicar como segredos devem ser tratados no GitHub Actions, incluindo escopo (repo vs environment), política de acesso e como evitar vazamento em logs.

Referências:
* https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets 
* https://docs.github.com/en/actions/concepts/security/secrets

Artefato esperado: ci.yml atualizado com login seguro e uso correto de secrets, além de comentários esclarecendo a proteção.

Questão 6 – Permissões mínimas do GITHUB_TOKEN e endurecimento do pipeline

Evolua o workflow definindo permissões explícitas e mínimas para o GITHUB_TOKEN, aumentando apenas onde necessário. O aluno deve explicar por que permissões excessivas ampliam a superfície de ataque e como o escopo por job reduz risco.

Referências:
* https://docs.github.com/en/actions/reference/security/secure-use e https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax

Artefato esperado: ci.yml com permissions globais restritivas e elevação pontual em job específico.

Questão 7 – SAST e falha controlada do pipeline para riscos críticos

Adicione um job de SAST com CodeQL para JavaScript. O job deve ser parte do gate antes do build/push de imagem. O aluno deve explicar o impacto de inserir segurança como gate “não opcional” e como tratar tempos de execução maiores sem comprometer o feedback.

Referências:
*  https://docs.github.com/en/actions (visão geral) e documentação do CodeQL via GitHub Docs (o aluno deve localizar a página relevante e citar no relatório do exercício)

Artefato esperado: ci.yml com job de CodeQL e dependências ajustadas.

Questão 8 – Separação CI/CD e promoção por ambiente com aprovação em produção

Crie um workflow separado de CD acionado em push na branch principal. Ele deve promover o artefato imutável (tag SHA) para homologação automaticamente e para produção apenas com aprovação manual via environment protegido. O aluno deve explicar por que a separação CI/CD melhora auditoria e por que environments são mais adequados do que “aprovação via script”.

Referências:
* https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments 
* https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments

Artefato esperado: criação de .github/workflows/cd.yml usando environment: staging e environment: production.

Questão 9 – Compartilhamento de artefatos e evidências do pipeline

Adicione ao CI a publicação de evidências como artefatos de workflow, por exemplo relatórios de teste, coverage ou relatórios de lint. O aluno deve usar upload/download de artefatos para compartilhar dados entre jobs e explicar por que isso melhora diagnóstico e governança.

Referências: 
* https://docs.github.com/en/actions/tutorials/store-and-share-data e https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts

Artefato esperado: ci.yml com upload-artifact e, quando aplicável, download-artifact.

Questão 10 – Generalização com workflows reutilizáveis

Refatore parte do pipeline para um workflow reutilizável acionado por workflow_call, permitindo que outros repositórios ou serviços reutilizem a automação sem duplicação. O aluno deve definir inputs e secrets e demonstrar chamada a esse workflow.

Referências:
* https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows e 
* https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations

Artefato esperado: criação de .github/workflows/_reusable-node-ci.yml e ajuste do workflow chamador.

