O gabarito repete o enunciado de cada questão com formatação diferenciada, e em seguida apresenta a resposta esperada com YAML incremental e comentado. Os trechos são propostos para serem diretamente comparáveis com o que o aluno implementou.

## Questão 1 – Resposta

Enunciado: Workflow mínimo de Lint como primeiro gate.

> Implemente um workflow de CI acionado em pull requests para a branch principal, contendo um job exclusivo para lint. O job deve fazer checkout do repositório, configurar o Node.js (LTS), instalar dependências com reprodutibilidade e executar o lint.

Resposta: o workflow deve rodar em pull requests para evitar integração de código com violações de padrão. O YAML abaixo cria um job único de lint, com passos reprodutíveis.

Arquivo: .github/workflows/ci.yml
```yaml 
name: CI

on:
  # Executa em PRs para a branch principal, protegendo a linha de integração.
  pull_request:
    branches: [ main ]


jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      # Faz checkout do código para o runner.
      - name: Checkout
        uses: actions/checkout@v4

      # Configura Node LTS para padronizar o runtime do pipeline.
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      # npm ci garante instalação reprodutível com base no lockfile.
      - name: Install dependencies
        run: npm ci

      # Gate de estilo e padrões antes de qualquer outro processamento.
      - name: Run lint
        run: npm run lint
```

Questão 2 – Resposta

Enunciado: Adição de testes automatizados em job separado e dependência explícita.

> Evolua o workflow criando um job separado para testes automatizados. O job de testes deve depender do sucesso do lint, de forma explícita, usando dependências entre jobs. O aluno deve justificar por que o lint é colocado antes dos testes neste caso.

Resposta: a separação em jobs aumenta clareza e permite evoluções futuras. O needs garante que testes só rodem se lint passar, evitando desperdício de execução quando há falhas triviais de padronização.

Arquivo: .github/workflows/ci.yml (incremento)

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Install dependencies
        run: npm ci
      - name: Run lint
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    # Depende explicitamente do sucesso do lint.
    needs: lint

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test
```

Questão 3 – Resposta

Enunciado: Otimização com cache de dependências e redução do tempo de feedback.

> Evolua o workflow para reduzir o tempo total de execução. Aplique cache de dependências do Node, de forma compatível com npm ci, mantendo consistência entre jobs. Explique o que pode dar errado se o cache for configurado de forma inadequada.

Resposta: o cache reduz tempo de download/instalação. A forma mais simples no ecossistema Node é usar o cache embutido do actions/setup-node. O cuidado é alinhar cache com lockfile; mudanças no package-lock.json devem invalidar o cache de forma natural.

Arquivo: .github/workflows/ci.yml (incremento)

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          # Cache de dependências para acelerar execuções repetidas.
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

Questão 4 – Resposta

Enunciado: Build de imagem Docker como artefato imutável após gates de qualidade.

> Adicione um job de build de imagem Docker que só execute após lint e testes passarem. A imagem deve ser versionada com tag imutável baseada no SHA do commit. O aluno deve explicar como essa decisão habilita rastreabilidade e promoção do mesmo artefato nos deploys.



Resposta: o build deve ocorrer somente após gates passarem. A tag github.sha permite rastrear exatamente qual commit gerou a imagem. Essa mesma tag deve ser usada no CD para garantir promoção sem rebuild.

Arquivo: .github/workflows/ci.yml (incremento)

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myorg/myapp

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run lint
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  docker-build:
    runs-on: ubuntu-latest
    # Build só após gates de qualidade.
    needs: [lint, test]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Neste ponto apenas construímos a imagem localmente.
      # O push para registry será tratado após implementar segredos.
      - name: Build Docker image (local)
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker images | head
```

Questão 5 – Resposta

Enunciado: Gestão de segredos e princípio do menor privilégio.

> Introduza no pipeline a necessidade de autenticação em um registry privado (ou no GitHub Container Registry). O aluno deve implementar login seguro e explicar como segredos devem ser tratados no GitHub Actions, incluindo escopo (repo vs environment), política de acesso e como evitar vazamento em logs.

Resposta: a autenticação no registry deve usar secrets, nunca valores em texto plano. Em GHCR é comum usar GITHUB_TOKEN, mas ainda assim deve-se restringir permissões e evitar imprimir variáveis em logs. Se a organização usar token dedicado, ele deve ser armazenado como secret de repositório ou, preferencialmente, como secret de environment quando o acesso depender de aprovação.

Arquivo: .github/workflows/ci.yml (incremento)

```yaml
name: CI
on:
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myorg/myapp

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run lint
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  docker-build-and-push:
    runs-on: ubuntu-latest
    needs: [lint, test]

    # Evita publicar imagens a partir de PRs de forks.
    if: ${{ github.event.pull_request.head.repo.full_name == github.repository }}

    # Para publicar em registry, precisamos elevar permissões apenas neste job.
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Login no GHCR usando GITHUB_TOKEN.
      # Alternativa corporativa: usar um PAT armazenado em secrets (ex.: secrets.REGISTRY_TOKEN).
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Build e push com tag imutável.
      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

Questão 6 – Resposta

Enunciado: Permissões mínimas do GITHUB_TOKEN e endurecimento do pipeline.

> Evolua o workflow definindo permissões explícitas e mínimas para o GITHUB_TOKEN, aumentando apenas onde necessário. O aluno deve explicar por que permissões excessivas ampliam a superfície de ataque e como o escopo por job reduz risco.

Resposta: o pipeline deve ser “read-only por padrão” e elevar permissões apenas onde necessário. Isso é especialmente importante quando workflows podem ser disparados por PRs, onde há risco ampliado de execução com código não confiável.

Arquivo: .github/workflows/ci.yml (incremento)

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

# Default: mínimo necessário para checkout.
permissions:
  contents: read

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myorg/myapp

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run lint
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  docker-build-and-push:
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: ${{ github.event.pull_request.head.repo.full_name == github.repository }}

    # Elevação pontual: apenas aqui precisamos escrever packages.
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

Questão 7 – Resposta

Enunciado: SAST e falha controlada do pipeline para riscos críticos.

> Adicione um job de SAST com CodeQL para JavaScript. O job deve ser parte do gate antes do build/push de imagem. O aluno deve explicar o impacto de inserir segurança como gate “não opcional” e como tratar tempos de execução maiores sem comprometer o feedback.

Resposta: CodeQL entra como gate não opcional antes de build/push, para impedir que vulnerabilidades conhecidas avancem. Como a análise pode ser mais lenta, costuma-se manter lint/test rápidos para feedback inicial e executar SAST em paralelo, mas ainda como requisito para liberar artefato.

Arquivo: .github/workflows/ci.yml (incremento)

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

permissions:
  contents: read

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myorg/myapp

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run lint
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  codeql:
    runs-on: ubuntu-latest

    # Permissões específicas para publicar eventos de segurança.
    permissions:
      contents: read
      security-events: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Analyze
        uses: github/codeql-action/analyze@v3

  docker-build-and-push:
    runs-on: ubuntu-latest
    # Agora o build/push só ocorre se CodeQL também passou.
    needs: [lint, test, codeql]
    if: ${{ github.event.pull_request.head.repo.full_name == github.repository }}

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

Questão 8 – Resposta

Enunciado: Separação CI/CD e promoção por ambiente com aprovação em produção.

> Crie um workflow separado de CD acionado em push na branch principal. Ele deve promover o artefato imutável (tag SHA) para homologação automaticamente e para produção apenas com aprovação manual via environment protegido. O aluno deve explicar por que a separação CI/CD melhora auditoria e por que environments são mais adequados do que “aprovação via script”.

Resposta: o CD deve ser um workflow separado, acionado por push em main, e deve promover o artefato imutável gerado a partir do commit. O uso de environments delega a aprovação para o mecanismo de proteção da plataforma, reduzindo “gambiarras” com scripts e aumentando auditabilidade.


Arquivo: .github/workflows/cd.yml
```yaml
name: CD

on:
  push:
    branches: [ main ]

permissions:
  contents: read

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myorg/myapp
  IMAGE_TAG: ${{ github.sha }}

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    # Em GitHub, o environment pode impor regras de proteção e segredos por ambiente.
    environment: staging

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          ./scripts/deploy.sh staging "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}"

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    # Environment protegido com required reviewers habilita aprovação manual.
    environment: production

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy to production
        run: |
          ./scripts/deploy.sh production "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}"
```

Questão 9 – Resposta

Enunciado: Compartilhamento de artefatos e evidências do pipeline.

> Adicione ao CI a publicação de evidências como artefatos de workflow, por exemplo relatórios de teste, coverage ou relatórios de lint. O aluno deve usar upload/download de artefatos para compartilhar dados entre jobs e explicar por que isso melhora diagnóstico e governança.

Resposta: publicar evidências como artefatos permite inspeção pós-execução, facilita auditoria e acelera diagnóstico. O aluno deve configurar o teste para gerar um relatório (por exemplo, JUnit/coverage) e então fazer upload do diretório. O lint também pode produzir relatório em formato consumível.

Arquivo: .github/workflows/ci.yml (incremento exemplificando upload no job de testes)

```yaml

# Trecho do job 'test' com upload de evidências.

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Node (with npm cache)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci

      # O comando abaixo depende do projeto estar configurado para gerar relatórios.
      - name: Run tests (with reports)
        run: |
          npm test -- --ci

      # Publica evidências do pipeline para análise posterior.
      - name: Upload test artifacts
        uses: actions/upload-artifact@v4
        with:
          name: test-evidence
          path: |
            coverage/
            test-results/
          if-no-files-found: warn
```

Questão 10 – Resposta

Enunciado: Generalização com workflows reutilizáveis.

> Refatore parte do pipeline para um workflow reutilizável acionado por workflow_call, permitindo que outros repositórios ou serviços reutilizem a automação sem duplicação. O aluno deve definir inputs e secrets e demonstrar chamada a esse workflow.

Resposta: a reutilização evita duplicação quando a empresa possui múltiplos serviços. O workflow reutilizável define workflow_call, expõe inputs e secrets e centraliza padrões. O workflow chamador passa parâmetros, mantendo governança e consistência.

Arquivo: .github/workflows/_reusable-node-ci.yml
```yaml
name: Reusable Node CI

on:
  workflow_call:
    inputs:
      node_version:
        required: true
        type: string
      run_tests:
        required: false
        type: boolean
        default: true
    secrets:
      REGISTRY_TOKEN:
        required: false

permissions:
  contents: read

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node_version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      # Execução condicional de testes, controlada por input.
      - name: Test
        if: ${{ inputs.run_tests }}
        run: npm test
```

Arquivo chamador: .github/workflows/ci.yml (exemplo de chamada)

```yaml
name: CI

on:
  pull_request:
    branches: [ main ]

jobs:
  call-reusable:
    uses: ./.github/workflows/_reusable-node-ci.yml
    with:
      node_version: '20'
      run_tests: true
````
