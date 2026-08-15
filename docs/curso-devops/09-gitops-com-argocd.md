# Aula 09 — GitOps com Argo CD

[⬅ Aula anterior](08-helm-empacotando-a-aplicacao.md) | [Índice](00-indice.md) | [Próxima aula ➡](10-ci-com-github-actions.md)

## Objetivos

- Instalar o Argo CD no cluster e entender seus componentes principais.
- Entender o objeto `Application` e como ele conecta Git ao cluster.
- Entender `syncPolicy` (automated, prune, selfHeal) e o que cada opção realmente faz.
- Expor a UI do Argo CD via Gateway API, reaproveitando o que foi visto na aula 06.

## Conceito

### Relembrando GitOps (aula 01)

O Argo CD é o **agente de reconciliação** que vive dentro do cluster e implementa o padrão
GitOps: ele observa um repositório Git e garante continuamente que o estado do cluster seja
igual ao que está descrito lá.

```mermaid
flowchart LR
    Dev["Desenvolvedor"] -->|git push| Repo["Repositório Git\n(este projeto)"]
    Repo -->|Argo CD faz PULL\n(não é push de fora)| ArgoCD["Argo CD\n(dentro do cluster)"]
    ArgoCD -->|aplica/reconcilia| Cluster["Cluster Kubernetes"]
    Cluster -.se alguém mudar manualmente\nselfHeal desfaz.-> ArgoCD
```

Note a seta: quem inicia a sincronização é o **Argo CD** (pull), não um pipeline de CD externo
empurrando `kubectl apply` (push). Essa diferença tem uma consequência de segurança
importante: o cluster **não precisa dar credenciais de escrita** para nenhum sistema de CI/CD
externo — só o Argo CD, que já vive dentro do cluster, precisa de permissão para aplicar
manifests.

### Componentes do Argo CD

- **API/Server**: UI web + API usada pelo `argocd` CLI.
- **Repo Server**: clona o Git, renderiza manifests (Helm, Kustomize, YAML puro).
- **Application Controller**: compara o estado desejado (Git) com o estado real (cluster) e
  aplica as diferenças.
- **Redis / Dex**: cache e autenticação (SSO), respectivamente.

### O objeto `Application`

Um `Application` é a unidade central: aponta **de onde** vem o manifest (`source`) e **para
onde** ele vai (`destination`).

O arquivo real deste projeto,
[argocd/argocd-apps/boutique-app.yaml](../../argocd/argocd-apps/boutique-app.yaml):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: boutique-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/laxmikantagiri/Production-Grade_GitOps-Driven_Microservices-Demo.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: boutique-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

- `source.path: "."` → aponta para a **raiz do repositório**, onde está o
  [kustomization.yaml](../../kustomization.yaml) que você estudou na aula 07 (o que combina o
  chart Helm com os manifests extras de rede). O Argo CD detecta automaticamente que é um
  projeto Kustomize e o processa.
- `destination.namespace: boutique-app` → tudo é instalado nesse namespace.
- `syncPolicy.automated.prune: true` → recursos removidos do Git são removidos do cluster.
- `syncPolicy.automated.selfHeal: true` → mudanças manuais no cluster (`kubectl edit`, etc.)
  são revertidas automaticamente para bater com o Git.
- `syncOptions: [CreateNamespace=true]` → cria o namespace `boutique-app` se não existir.

### Habilitando Helm dentro do Kustomize no Argo CD

Como visto na aula 07, o suporte a Helm dentro do Kustomize é um "plugin inseguro" desabilitado
por padrão. Em [argocd/argocd-values-9.4.0.yaml](../../argocd/argocd-values-9.4.0.yaml), isso
é habilitado explicitamente:

```yaml
configs:
  cm:
    create: true
    kustomize.buildOptions: "--enable-helm"
```

### TLS/roteamento do próprio Argo CD

Como o TLS é terminado no ALB (Gateway API, aula 06), o servidor do Argo CD roda internamente
em modo "inseguro" (HTTP simples entre o ALB e o Pod — o tráfego externo continua criptografado
via HTTPS no listener do `Gateway`):

```yaml
server:
  insecure: true
```

E o próprio Argo CD se expõe usando o mesmo padrão `HTTPRoute` +
`TargetGroupConfiguration` que você já viu na aula 06:

```yaml
server:
  httproute:
    enabled: true
    parentRefs:
      - name: app-alb-gateway
        namespace: default
        sectionName: https
    hostnames:
      - argocd.devopsdock.site
```

`argocd/target-grp-config.yaml` complementa com `targetType: ip`.

## Na prática, neste repositório

### 1. Instalar o Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-cd argo/argo-cd -n argocd -f argocd/argocd-values-9.4.0.yaml \
  --version 9.4.0 --create-namespace

kubectl apply -f argocd/target-grp-config.yaml
```

### 2. Login

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# usuário: admin
```

Acesse `https://argocd.devopsdock.site` (ou troque pelo seu domínio) e troque a senha padrão.

### 3. Criar a Application

```bash
kubectl apply -f argocd/argocd-apps/boutique-app.yaml
```

Na UI do Argo CD, você verá o app `boutique-app` sincronizando e todos os recursos
(Deployments, Services, HTTPRoute, TargetGroupConfiguration) aparecendo como uma árvore visual.

### 4. Comprove o `selfHeal` na prática

```bash
kubectl scale deployment frontend -n boutique-app --replicas=5
# espere alguns segundos e rode de novo:
kubectl get deployment frontend -n boutique-app
```

Você verá o número de réplicas **voltar sozinho** para o que está definido no
`values.yaml`/chart — o Argo CD detectou o drift e corrigiu. Esse comportamento é exatamente o
que a aula 14 (HPA) vai te lembrar: mudanças devem ir para o Git, nunca direto no cluster.

## Checklist

- [ ] Eu sei explicar por que GitOps é "pull" e não "push".
- [ ] Eu sei o que `prune` e `selfHeal` fazem, com um exemplo prático de cada.
- [ ] Eu sei por que o Argo CD precisa de `kustomize.buildOptions: --enable-helm` neste projeto.
- [ ] Eu instalei o Argo CD e apliquei a `Application` `boutique-app` com sucesso.

## Próxima aula

[Aula 10 — CI com GitHub Actions ➡](10-ci-com-github-actions.md)
