# Aula 14 — Escalabilidade: Horizontal Pod Autoscaler (HPA)

[⬅ Aula anterior](13-observabilidade-logs-eck-kibana.md) | [Índice](00-indice.md) | [Próxima aula ➡](15-service-mesh-istio.md)

## Objetivos

- Entender a diferença entre escalar verticalmente e horizontalmente.
- Entender como o HPA decide quando e quanto escalar.
- Configurar o `metrics-server` (pré-requisito do HPA) e o HPA do `frontend`.
- Entender por que, em um ambiente GitOps, HPA não pode ser ajustado via `kubectl edit`.

## Conceito

### Escalar vertical vs. horizontalmente

- **Escalar verticalmente**: dar mais CPU/memória a uma única instância. Tem limite físico e
  geralmente exige reiniciar o processo.
- **Escalar horizontalmente**: rodar **mais réplicas** idênticas do mesmo serviço, dividindo a
  carga entre elas. É o que containers/Kubernetes fazem bem, especialmente para serviços
  **stateless** (lembra da aula 02? é exatamente por isso que a maioria dos microsserviços
  deste projeto é stateless — fica trivial rodar 1, 5 ou 20 réplicas do `frontend` sem se
  preocupar com dados).

O **Horizontal Pod Autoscaler (HPA)** automatiza essa segunda estratégia: observa uma métrica
(normalmente CPU) e ajusta o número de réplicas de um Deployment dentro de um intervalo
`minReplicas`/`maxReplicas`.

### Pré-requisito: o `metrics-server`

O HPA (e o comando `kubectl top`) não funcionam sem uma fonte de métricas de uso real de
CPU/memória por Pod. O **metrics-server** é um componente leve que coleta essas métricas dos
`kubelet`s de cada nó e as expõe pela API de métricas do Kubernetes (`metrics.k8s.io`), que o
HPA consulta.

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server --version 3.13.0 -n kube-system

kubectl top nodes
kubectl top pods -n boutique-app
```

### O HPA deste projeto

[scaling/frontend-hpa.yaml](../../scaling/frontend-hpa.yaml):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: boutique-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 1
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 5
```

- `scaleTargetRef` → qual Deployment o HPA controla (`frontend`).
- `minReplicas`/`maxReplicas` → nunca menos que 1, nunca mais que 6 réplicas.
- `averageUtilization: 5` → o HPA calcula a **porcentagem média de uso de CPU em relação ao
  `requests.cpu`** (lembra da aula 04: `requests` é a base do cálculo!) de todos os Pods do
  Deployment, e tenta manter essa média em torno de 5%.

> [!IMPORTANTE]
> `5%` é **propositalmente baixo**. Em produção real, um valor comum seria `50%`-`70%`. Aqui,
> o valor baixo garante que **qualquer tráfego mínimo** (mesmo de teste) já dispare o
> autoscaling rapidamente, o que é ótimo para **aprender e demonstrar** o mecanismo — mas seria
> um desperdício de recursos (réplicas demais para pouca carga real) em um ambiente de
> produção de verdade. Sempre leia os valores de configuração perguntando "isso é para
> demonstração ou para produção?".

O cálculo aproximado que o HPA faz:

$$\text{réplicas desejadas} = \left\lceil \text{réplicas atuais} \times \frac{\text{uso atual de CPU (\%)}}{\text{alvo (\%)}} \right\rceil$$

### GitOps + HPA: por que `kubectl edit` não funciona aqui

Este é um dos pontos mais importantes do curso, revisitando a aula 09: como a aplicação é
gerida pelo Argo CD com `selfHeal: true`, qualquer alteração feita direto no cluster (seja em
réplicas do Deployment, seja em variáveis de ambiente do `loadgenerator` para gerar mais
carga) é **revertida automaticamente** pelo Argo CD, porque ele detecta que o cluster "saiu da
linha" em relação ao Git.

**Processo correto em GitOps:**

1. Edite o valor no `values.yaml` (ou no manifest do HPA, se ele for versionado no chart).
2. `git commit` + `git push`.
3. Deixe o Argo CD sincronizar sozinho.

## Na prática, neste repositório

### 1. Aplicar o HPA e observar

```bash
kubectl apply -f scaling/frontend-hpa.yaml
kubectl get hpa -n boutique-app -w
```

Em um segundo terminal:

```bash
kubectl get pods -n boutique-app -w
```

Como o `loadgenerator` já está gerando tráfego constante (aula 02), e o alvo está em apenas
5%, você deve ver o número de réplicas do `frontend` subir sozinho em poucos minutos.

### 2. Validar a métrica manualmente

```bash
kubectl top pods -n boutique-app
```

Se a CPU do `frontend` ultrapassar 5% do `requests.cpu` definido (100m, conforme a aula 04),
o HPA aumenta réplicas.

### 3. Cruzar com observabilidade (aulas 12/13)

No Kibana, filtre por `kubernetes.deployment.name: "frontend"` para ver os logs dos novos Pods
aparecendo. No Grafana, você pode montar um painel mostrando a contagem de réplicas ao longo
do tempo.

### 4. Repita para outros serviços

O README sugere aplicar HPAs semelhantes para `cartservice`, `checkoutservice` e
`recommendationservice` — os serviços mais chamados no caminho crítico de uma compra (aula
02). Copie a estrutura de `frontend-hpa.yaml`, trocando apenas `metadata.name` e
`scaleTargetRef.name`.

## Checklist

- [ ] Eu sei a diferença entre escalar vertical e horizontalmente.
- [ ] Eu sei por que o HPA depende do `metrics-server`.
- [ ] Eu sei calcular, dado `requests.cpu` e uso atual, se o HPA vai aumentar réplicas.
- [ ] Eu sei explicar por que `kubectl edit` em um Deployment gerido por Argo CD com
      `selfHeal: true` não funciona, e qual é o processo correto.

## Próxima aula

[Aula 15 — Service Mesh com Istio ➡](15-service-mesh-istio.md)
