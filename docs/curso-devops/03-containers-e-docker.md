# Aula 03 — Containers e Docker

[⬅ Aula anterior](02-arquitetura-da-aplicacao.md) | [Índice](00-indice.md) | [Próxima aula ➡](04-fundamentos-kubernetes.md)

## Objetivos

- Entender o que é um container e por que ele resolve o "funciona na minha máquina".
- Entender a diferença entre imagem e container, e o papel de um `Dockerfile`.
- Entender como cada microsserviço deste projeto vira uma imagem versionada.
- Saber construir, taguear e publicar uma imagem em um registry (GHCR).

## Conceito

### O problema que containers resolvem

Antes de containers, "implantar" um serviço significava garantir que o servidor de produção
tivesse exatamente a mesma versão de linguagem, bibliotecas e configuração do ambiente de
desenvolvimento. Pequenas diferenças causavam bugs difíceis de reproduzir.

Um **container** empacota a aplicação **junto com tudo que ela precisa para rodar**
(runtime, bibliotecas, variáveis de ambiente, arquivos) em uma unidade isolada e portátil.
O mesmo container roda igual no laptop do desenvolvedor, no CI e em produção.

### Imagem vs. container

- **Imagem**: um pacote imutável (somente leitura) com o sistema de arquivos da aplicação —
  o "molde". Construída a partir de um `Dockerfile`.
- **Container**: uma **instância em execução** dessa imagem — o "objeto" criado a partir do
  molde. Você pode rodar vários containers a partir da mesma imagem.

Um `Dockerfile` é uma receita, passo a passo, para montar a imagem. Por exemplo, um Dockerfile
típico Go (parecido com o do `frontend`) segue o padrão **multi-stage build**:

```dockerfile
# Estágio 1: compila o binário Go
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /server .

# Estágio 2: imagem final, enxuta, só com o binário
FROM gcr.io/distroless/base
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

Isso mantém a imagem final pequena e com menos superfície de ataque (sem compilador, sem
código-fonte) — importante para segurança (ver aula 16).

### Cada microsserviço = uma imagem independente

Como vimos na aula 02, cada pasta em [src/](../../src) é autocontida: tem seu próprio
`Dockerfile`. Isso significa:

- O `frontend` (Go) e o `emailservice` (Python) são buildados **de formas completamente
  diferentes**, cada um usando a imagem base certa para sua linguagem.
- Cada serviço tem **seu próprio ciclo de vida de versão** — você pode atualizar só o
  `paymentservice` sem tocar nos outros 10.
- Isso é o que permite ao pipeline de CI (aula 10) buildar **apenas o serviço que mudou** em
  vez da aplicação inteira.

### Registro de imagens (Container Registry)

Depois de construída, a imagem precisa ficar em algum lugar acessível pelo cluster Kubernetes:
um **registry**. Este projeto usa o **GitHub Container Registry (GHCR)**, publicando imagens
como:

```
ghcr.io/laxmikantagiri/microservices-demo/<nome-do-serviço>:<tag>
```

Isso é visível diretamente no Helm chart, em
[helm-chart/values.yaml](../../helm-chart/values.yaml):

```yaml
images:
  frontend:
    repository: ghcr.io/laxmikantagiri/microservices-demo/frontend
    tag: v0.10.4
  cartservice:
    repository: ghcr.io/laxmikantagiri/microservices-demo/cartservice
    tag: v0.10.4
```

Note que a **tag** da imagem (`v0.10.4`, ou depois `sha-<commit>` quando o CI assume o
controle — aula 10/11) é o elo entre "o que o CI construiu" e "o que o Kubernetes vai rodar".

## Na prática, neste repositório

### 1. Build local de uma imagem

```bash
cd src/emailservice
docker build -t meu-usuario/emailservice:teste .
```

### 2. Rodar o container localmente

```bash
docker run --rm -p 8080:8080 meu-usuario/emailservice:teste
```

### 3. Login e push para o GHCR (documentado no FAQ do [README.md](../../README.md))

```bash
echo <SEU_TOKEN> | docker login ghcr.io -u SEU_USUARIO --password-stdin

docker tag meu-usuario/emailservice:teste ghcr.io/SEU_USUARIO/microservices-demo/emailservice:v1

docker push ghcr.io/SEU_USUARIO/microservices-demo/emailservice:v1
```

> [!IMPORTANTE]
> O token precisa da permissão `packages: read & write` (e `contents: read` se o repositório
> for privado). É o mesmo tipo de credencial usada pelo pipeline de CI na aula 10 — só que lá
> quem faz login é o `GITHUB_TOKEN` automático do Actions, não uma pessoa.

### 4. Explore os scripts de release do projeto

A pasta [docs/releasing/](../releasing) tem scripts que automatizam esse processo para todos
os 11 serviços de uma vez:

- [make-docker-images.sh](../releasing/make-docker-images.sh) — builda e publica todas as
  imagens.
- [make-helm-chart.sh](../releasing/make-helm-chart.sh) — empacota o Helm chart (aula 08).
- [make-release-artifacts.sh](../releasing/make-release-artifacts.sh) e
  [make-release.sh](../releasing/make-release.sh) — geram os artefatos de uma release
  completa.

Vale a pena abrir esses scripts e ler linha a linha: eles mostram, em forma de automação, tudo
que você acabou de fazer manualmente acima.

## Checklist

- [ ] Eu sei explicar a diferença entre imagem e container.
- [ ] Eu sei o que é um multi-stage build e por que ele deixa a imagem final menor/mais segura.
- [ ] Eu sei buildar e taguear uma imagem localmente com `docker build`/`docker tag`.
- [ ] Eu sei onde, no Helm chart deste projeto, fica definido o repositório/tag de cada imagem.

## Próxima aula

[Aula 04 — Fundamentos de Kubernetes ➡](04-fundamentos-kubernetes.md)
