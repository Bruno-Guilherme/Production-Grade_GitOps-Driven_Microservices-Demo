# Aula 08 — Helm: empacotando a aplicação

[⬅ Aula anterior](07-kustomize-variacoes-de-deploy.md) | [Índice](00-indice.md) | [Próxima aula ➡](09-gitops-com-argocd.md)

## Objetivos

- Entender o problema que o Helm resolve: parametrizar e distribuir aplicações Kubernetes.
- Entender a estrutura de um chart: `Chart.yaml`, `values.yaml`, `templates/`.
- Ler o chart real deste projeto (`helm-chart/`) e entender seus templates.
- Empacotar e publicar o chart como artefato OCI no GHCR.

## Conceito

### O problema: reaplicar 11 manifests toda vez é repetitivo e frágil

Você já viu (aula 04) que cada serviço tem Deployment + Service + ServiceAccount. Multiplique
por 11 serviços: são dezenas de arquivos YAML quase idênticos, cada um com pequenas
diferenças (nome, porta, recursos, réplicas). Sem um mecanismo de templating, qualquer mudança
estrutural (ex.: adicionar uma label em todos os Deployments) significa editar 11 arquivos.

**Helm** é o "gerenciador de pacotes" do Kubernetes: você define **templates** (com variáveis)
uma única vez, e um arquivo `values.yaml` com os valores que mudam por ambiente/serviço. O
Helm gera o YAML final combinando os dois — e versiona/empacota tudo isso como um **chart**.

### Anatomia de um chart

```
helm-chart/
├── Chart.yaml       # metadados: nome, versão do chart, versão da aplicação
├── values.yaml      # valores padrão (podem ser sobrescritos na instalação)
└── templates/       # manifests com sintaxe Go template ({{ .Values.x }})
```

[helm-chart/Chart.yaml](../../helm-chart/Chart.yaml):

```yaml
apiVersion: v2
name: onlineboutique
version: 0.10.4      # versão do CHART (a "embalagem")
appVersion: "v0.10.4" # versão da APLICAÇÃO dentro dele
```

Repare que `version` (do chart) e `appVersion` (da aplicação) são conceitos diferentes: o
chart em si tem seu próprio versionamento semântico, independente da versão do app que ele
empacota.

### `values.yaml`: o painel de controle da aplicação

[helm-chart/values.yaml](../../helm-chart/values.yaml) centraliza tudo que pode variar entre
instalações — sem precisar tocar nos templates:

```yaml
images:
  frontend:
    repository: ghcr.io/laxmikantagiri/microservices-demo/frontend
    tag: v0.10.4
  cartservice:
    repository: ghcr.io/laxmikantagiri/microservices-demo/cartservice
    tag: v0.10.4
# ...

networkPolicies:
  create: false        # liga/desliga NetworkPolicy globalmente

authorizationPolicies:
  create: false         # liga/desliga AuthorizationPolicy (Istio) globalmente

frontend:
  create: true
  name: frontend
  resources:
    requests: { cpu: 100m, memory: 64Mi }
    limits:   { cpu: 200m, memory: 128Mi }
```

Cada serviço tem sua própria seção (`adService`, `cartService`, `checkoutService`...) com
`resources` próprios — isso é o que a aula 04 mostrou "hardcoded" no manifest puro; aqui vira
**parametrizável**.

### Templates: onde a "mágica" acontece

Um exemplo real e simples, [helm-chart/templates/common.yaml](../../helm-chart/templates/common.yaml):

```yaml
{{- if .Values.networkPolicies.create }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: {{ .Release.Namespace }}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
{{- end }}
{{- if .Values.authorizationPolicies.create }}
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: {{ .Release.Namespace }}
spec: {}
{{- end }}
```

Isso ilustra os dois recursos essenciais de templating do Helm:

- **Condicionais** (`{{- if .Values.x }}`): o recurso só é gerado se o valor for `true` —
  é assim que features opcionais (NetworkPolicies, AuthorizationPolicies, OpenTelemetry
  Collector, Google Cloud Operations) são ligadas/desligadas sem editar templates.
- **Interpolação de variáveis** (`{{ .Release.Namespace }}`): o Helm injeta valores do
  contexto de instalação (namespace, nome do release) automaticamente.

Explore também [helm-chart/templates/frontend.yaml](../../helm-chart/templates/frontend.yaml)
e compare com o manifest puro da aula 04 — você vai ver a mesma estrutura, só que com
`{{ .Values.frontend.resources... }}` no lugar dos valores fixos.

### Empacotando e distribuindo (Helm + OCI)

Charts modernos podem ser publicados como artefatos **OCI** (o mesmo formato usado por
imagens de container!) direto em registries como o GHCR — sem precisar de um "Helm repo"
tradicional separado.

```bash
cd helm-chart/
helm package .
# gera onlineboutique-0.10.4.tgz

echo <TOKEN> | helm registry login ghcr.io -u SEU_USUARIO --password-stdin
helm push onlineboutique-0.10.4.tgz oci://ghcr.io/SEU_USUARIO
```

A partir daí, qualquer pessoa (ou o ArgoCD, na próxima aula) pode instalar diretamente:

```bash
helm install boutique-app oci://ghcr.io/laxmikantagiri/onlineboutique --version 0.10.4
```

> [!IMPORTANTE]
> É exatamente essa referência OCI (`oci://ghcr.io/laxmikantagiri`, versão `0.10.4`) que
> aparece no `helmCharts` do [kustomization.yaml](../../kustomization.yaml) raiz que você viu
> na aula 07. O chart Helm é a "unidade de distribuição" da aplicação; o Kustomize é quem o
> consome, combinando com manifests extras.

O script [docs/releasing/make-helm-chart.sh](../releasing/make-helm-chart.sh) automatiza esse
empacotamento como parte do processo de release do projeto.

## Na prática, neste repositório

1. Renderize o chart localmente, sem instalar nada, para ver o YAML final:
   ```bash
   helm template boutique-app helm-chart/ -n boutique-app
   ```
2. Altere um valor (ex.: `frontend.resources.limits.cpu`) em uma cópia local do
   `values.yaml` e rode `helm template` de novo — veja o efeito imediato no manifest gerado.
3. Leia [helm-chart/README.md](../../helm-chart/README.md) e
   [helm-chart/templates/NOTES.txt](../../helm-chart/templates/NOTES.txt) (a mensagem exibida
   após `helm install`).

## Checklist

- [ ] Eu sei a diferença entre `Chart.yaml` (metadados) e `values.yaml` (parâmetros).
- [ ] Eu sei ler uma condicional Helm (`{{- if .Values.x }}`) e uma interpolação
      (`{{ .Values.x }}`).
- [ ] Eu sei o que significa publicar um chart como artefato OCI e por que isso é conveniente.
- [ ] Eu entendo como o chart deste projeto se conecta ao `kustomization.yaml` raiz.

## Próxima aula

[Aula 09 — GitOps com ArgoCD ➡](09-gitops-com-argocd.md)
