# Aula 10 — CI com GitHub Actions

[⬅ Aula anterior](09-gitops-com-argocd.md) | [Índice](00-indice.md) | [Próxima aula ➡](11-cd-argocd-image-updater.md)

## Objetivos

- Entender o que é Integração Contínua (CI) e o que um pipeline de CI deve garantir.
- Entender workflows reutilizáveis do GitHub Actions e builds em matriz (matrix).
- Entender por que só os serviços alterados devem ser buildados (build seletivo).
- Entender scan de vulnerabilidades de imagem (Trivy) como parte do pipeline.

## Conceito

### O que é CI, de novo

CI é a prática de **integrar e validar** o código continuamente — cada `push` dispara um
processo automatizado que builda, testa e (aqui) empacota a aplicação, para detectar problemas
o quanto antes, em vez de descobri-los só quando alguém tentar rodar em produção.

Neste projeto, "buildar e empacotar" significa: **gerar uma imagem Docker versionada e
publicá-la no GHCR** — o artefato que o CD (aula 11) vai consumir depois.

### Por que build seletivo importa aqui

Este projeto tem **11 microsserviços independentes** (aula 02). Se cada `push` buildasse todas
as 11 imagens, o pipeline seria lento e desperdiçaria recursos de CI, mesmo quando você só
mudou uma linha no `emailservice`.

A solução: **detectar quais pastas em `src/` mudaram** e buildar **só essas**.

### Dois workflows que se complementam

O `README.md` deste projeto documenta dois arquivos de workflow (`.github/workflows/`):

**1. `ci-trigger.yaml`** — dispara em todo `push` na branch `main` que toque `src/**`, detecta
quais serviços mudaram e cria um build em **matrix** (um job paralelo por serviço alterado):

```yaml
on:
  push:
    branches: [ main ]
    paths:
      - "src/**"

jobs:
  detect-changes:
    steps:
      - name: Detect changed services
        run: |
          SERVICES=$(git diff --name-only ${{ github.event.before }} ${{ github.sha }} \
            | grep '^src/' | cut -d'/' -f2 | sort -u | jq -R -s -c 'split("\n")[:-1]')
          echo "services=$SERVICES" >> $GITHUB_OUTPUT

  build-and-push:
    needs: detect-changes
    if: needs.detect-changes.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
    uses: ./.github/workflows/microservice-ci.yaml
    with:
      service: ${{ matrix.service }}
```

A mágica está no `git diff --name-only`, que lista os arquivos alterados no push, filtra só os
que começam com `src/`, extrai o nome da pasta (`cut -d'/' -f2`) e monta uma lista JSON única
de serviços afetados — que vira a `matrix` do job seguinte.

**2. `microservice-ci.yaml`** — um **workflow reutilizável** (`workflow_call`), chamado uma vez
por serviço na matrix, que builda, escaneia e publica a imagem:

```yaml
on:
  workflow_call:
    inputs:
      service:
        required: true
        type: string

jobs:
  build:
    env:
      IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/microservices-demo/${{ inputs.service }}:sha-${{ github.sha }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build Image
        run: |
          docker build --cache-from=type=gha --cache-to=type=gha,mode=max \
            -t $IMAGE_NAME ./src/${{ inputs.service }}

      - name: Run Trivy Scan
        uses: aquasecurity/trivy-action@0.20.0
        with:
          scan-type: image
          image-ref: ${{ env.IMAGE_NAME }}
          severity: HIGH,CRITICAL
          exit-code: 0

      - name: Push Image
        run: docker push $IMAGE_NAME
```

Repare em alguns detalhes que valem entendimento:

- **Reusable workflow (`workflow_call`)**: em vez de duplicar os mesmos passos de build 11
  vezes, existe **um único** workflow parametrizado por `service`, chamado uma vez por item da
  matrix. É "DRY" (Don't Repeat Yourself) aplicado a pipelines.
- **Tag da imagem = `sha-<commit>`**: em vez de uma tag fixa como `latest` ou `v0.10.4`, cada
  build gera uma tag **única e rastreável** ligada ao commit exato. Isso é essencial para o
  Argo CD Image Updater da próxima aula conseguir identificar "qual é a imagem mais nova".
- **`GITHUB_TOKEN`**: um token automático e temporário gerado pelo próprio Actions — não é uma
  credencial de longa duração armazenada em segredo. É suficiente para autenticar no GHCR
  porque o repositório de pacotes (aula 03) foi configurado para aceitar esse token.
- **Cache do Buildx (`cache-from`/`cache-to type=gha`)**: reaproveita camadas de build entre
  execuções, acelerando builds subsequentes.

### O scan de segurança (Trivy)

O [Trivy](https://aquasecurity.github.io/trivy/) escaneia a imagem recém-construída em busca
de vulnerabilidades conhecidas (CVEs) em pacotes do SO e bibliotecas da aplicação, filtrando
por severidade `HIGH,CRITICAL`.

> [!IMPORTANTE]
> Neste pipeline, `exit-code: 0` significa que o scan **nunca falha o build**, mesmo
> encontrando vulnerabilidades críticas — ele só gera um relatório informativo. O próprio
> `README.md` do projeto reconhece isso como uma simplificação para fins didáticos e recomenda
> mudar para `exit-code: 1` em ambientes reais, o que faz o pipeline **parar** (e não publicar
> a imagem) se houver vulnerabilidade `HIGH`/`CRITICAL`. Vamos retomar esse ponto na aula 16
> (Segurança) com mais profundidade sobre **shift-left security** (levar verificações de
> segurança para o mais cedo possível no ciclo de desenvolvimento).

## Na prática, neste repositório

1. Crie os workflows (se ainda não existirem) em `.github/workflows/ci-trigger.yaml` e
   `.github/workflows/microservice-ci.yaml`, com o conteúdo mostrado acima (documentado
   também no [README.md](../../README.md) raiz).
2. Garanta que os pacotes do GHCR estejam vinculados ao repositório e com permissão de
   escrita (Package Settings → conectar repositório → "Write" access).
3. Faça uma alteração pequena em `src/emailservice/` (ex.: um comentário) e dê `git push` na
   `main`. Acompanhe a aba **Actions** do GitHub: você deve ver o job `detect-changes` seguido
   de **apenas um** job de build na matrix (`emailservice`), não os 11.
4. Verifique no GHCR que a nova imagem apareceu com a tag `sha-<hash do commit>`.

## Checklist

- [ ] Eu sei explicar por que usar `workflow_call` evita duplicação entre os 11 serviços.
- [ ] Eu sei como o pipeline detecta **quais** serviços mudaram (`git diff` + `paths`).
- [ ] Eu sei por que a tag da imagem é `sha-<commit>` em vez de uma tag fixa.
- [ ] Eu sei o que o Trivy faz e por que `exit-code: 0` é uma escolha (fraca) de segurança.

## Próxima aula

[Aula 11 — CD: fechando o loop com Argo CD Image Updater ➡](11-cd-argocd-image-updater.md)
