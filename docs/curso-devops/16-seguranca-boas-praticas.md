# Aula 16 — Segurança e boas práticas (revisão OWASP-style)

[⬅ Aula anterior](15-service-mesh-istio.md) | [Índice](00-indice.md) | [Próxima aula ➡](17-projeto-final-proximos-passos.md)

## Objetivos

- Consolidar, em um só lugar, todas as decisões de segurança já vistas nas aulas anteriores.
- Aprender a mapear práticas concretas de infraestrutura a categorias do OWASP Top 10 / boas
  práticas de segurança em nuvem.
- Identificar pontos de melhoria reais deste projeto (o próprio README reconhece alguns).

## Conceito

Segurança em DevOps ("DevSecOps") não é uma etapa final — é uma responsabilidade distribuída
em **cada camada** que você já estudou: rede, identidade, container, pipeline, segredo. Vamos
revisar o que este projeto já faz bem, e o que ficaria melhor em um ambiente real.

### 1. Rede: minimizar superfície de ataque

| Prática | Onde está | Aula |
|---|---|---|
| Endpoint da API do EKS **privado** (`endpoint_public_access = false`) | `terraform/eks.tf` | 05 |
| SSH do bastion liberado **só para o IP de quem aplica o Terraform** (`/32`, via `data.http.my_ip`) | `terraform/bastion-ec2.tf` | 05 |
| Subnets privadas para os nós do EKS; só o NAT Gateway tem saída direta | `terraform/vpc.tf` | 05 |
| TLS terminado no ALB, hostnames explícitos (`*.devopsdock.site`) em vez de aceitar qualquer host | `gateway-api-manifests/gateway.yaml` | 06 |
| `NetworkPolicy` opcional (`deny-all` por padrão, liberação explícita) | `helm-chart/templates/common.yaml`, component `network-policies` | 04, 07, 08 |
| Egress explícito e restrito em service mesh (`ServiceEntry` allowlist) | `istio-manifests/allow-egress-googleapis.yaml` | 15 |

**Ponto de atenção real**: o próprio bastion, mesmo com SSH restrito ao seu IP, é uma máquina
extra na sua superfície de ataque. Em ambientes mais maduros, prefere-se
[AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
(acesso sem porta SSH aberta e sem chave para gerenciar) no lugar de um bastion tradicional.

### 2. Identidade e acesso (IAM): menor privilégio

| Prática | Onde está |
|---|---|
| Política de IAM do External DNS restrita só às ações `route53:ChangeResourceRecordSets` / `ListResourceRecordSets` / `ListHostedZones` | `external-dns/policy.json` |
| EKS Pod Identity (identidade por `ServiceAccount`, sem chaves de acesso de longa duração espalhadas) | `terraform/eks.tf` (addon), aulas 06/13 |
| `enable_cluster_creator_admin_permissions` dá admin só a quem criou o cluster, não a todo mundo | `terraform/eks.tf` |

**Princípio central**: nunca dar `*`/admin quando uma lista específica de ações resolve. Antes
de copiar uma política de IAM de um tutorial, pergunte "quais ações este componente
realmente precisa?".

### 3. Containers: hardening do runtime

Relembrando a aula 04, o manifest do `frontend` já aplica várias práticas recomendadas pelo
CIS Benchmark para Kubernetes:

```yaml
securityContext:
  runAsNonRoot: true              # nunca como root
  runAsUser: 1000
containers:
  - securityContext:
      allowPrivilegeEscalation: false  # não pode ganhar mais privilégio que o pai
      capabilities:
        drop: ["ALL"]                  # remove capabilities Linux desnecessárias
      readOnlyRootFilesystem: true     # filesystem raiz imutável
```

Isso, combinado com imagens **multi-stage** e `distroless`/mínimas (aula 03), reduz
drasticamente o que um atacante conseguiria fazer mesmo comprometendo o processo da
aplicação (sem shell, sem gravar arquivos, sem escalar privilégio, sem capabilities extras).

### 4. Segredos: nunca em texto puro no Git

| Prática | Onde está |
|---|---|
| Webhook do Slack montado via `Secret` do Kubernetes, referenciado por `api_url_file`, nunca em `values.yaml` | `observability/helm-values/kube-prom-stack-*.yaml`, aula 12 |
| Senha do Argo CD/Grafana/Elastic gerada automaticamente e lida via `kubectl get secret` | aulas 09, 12, 13 |
| PAT do GitHub para GHCR usado como `GITHUB_TOKEN` efêmero do Actions, não hardcoded | `.github/workflows/microservice-ci.yaml`, aula 10 |
| Chave privada do bastion (`bastion-key.pem`) gerada **localmente**, não commitada | `terraform/bastion-ec2.tf`, aula 05 |

**Regra de ouro**: se um valor pode autenticar em algo (senha, token, chave privada, webhook),
ele nunca deve aparecer em texto puro em um arquivo versionado no Git — use `Secret` do
Kubernetes (idealmente criptografado em repouso, ou gerido por uma ferramenta como
[External Secrets Operator](https://external-secrets.io/) / AWS Secrets Manager em produção
real).

### 5. Pipeline de CI/CD: shift-left security

- **Trivy scan de imagem** a cada build (aula 10) — captura CVEs conhecidas antes de a imagem
  chegar ao registry.
- **Ponto de melhoria já identificado pelo próprio projeto**: `exit-code: 0` no Trivy hoje só
  gera relatório, não bloqueia. Em produção, mudar para `exit-code: 1` (falhar o pipeline em
  vulnerabilidades `HIGH`/`CRITICAL`) é o padrão recomendado — especialmente em setores
  regulados.
- **GitOps reduz a superfície do pipeline de CD**: como o Argo CD faz *pull* (aula 09), nenhum
  sistema de CI externo precisa de credenciais de escrita direta no cluster — o CI só precisa
  de permissão para publicar imagens (GHCR), nunca `kubeconfig` de produção.

### 6. Mapeamento rápido para OWASP Top 10 (aplicado a infraestrutura/APIs)

| Categoria OWASP (conceito) | Como este projeto mitiga |
|---|---|
| Quebra de controle de acesso | IAM de menor privilégio, `AuthorizationPolicy`/`NetworkPolicy` deny-by-default |
| Falhas criptográficas | TLS no ALB (aula 06), mTLS opcional via Istio (aula 15) |
| Configuração incorreta de segurança | `securityContext` restritivo por padrão, `endpoint_public_access=false` |
| Componentes vulneráveis/desatualizados | Scan Trivy no CI (aula 10), versões de charts/providers fixadas explicitamente |
| Falhas de identificação/autenticação | Segredos via `Secret`, não hardcoded; tokens efêmeros do GitHub Actions |
| Registro e monitoramento insuficientes | Stack completa de observabilidade + alertas no Slack (aulas 12/13) |

## Na prática, neste repositório

1. Faça uma "auditoria" rápida: para cada linha da tabela acima, abra o arquivo referenciado e
   confirme que a configuração realmente está lá (não confie só neste resumo — leia o
   original).
2. Proponha (e, se possível, implemente em um fork) pelo menos uma melhoria:
   - Mudar `exit-code: 0` para `exit-code: 1` no workflow de CI e testar o que acontece se uma
     imagem tiver uma CVE crítica.
   - Habilitar `networkPolicies.create: true` no Helm chart e validar que a comunicação entre
     serviços continua funcionando (ou identificar o que precisa ser liberado explicitamente).
   - Trocar o bastion por acesso via AWS Systems Manager Session Manager.

## Checklist

- [ ] Eu sei apontar, neste repositório, pelo menos uma prática de segurança em cada camada:
      rede, IAM, container, segredo, pipeline.
- [ ] Eu sei explicar por que `exit-code: 0` no Trivy é uma fraqueza intencionalmente
      documentada, e o que mudar para corrigir.
- [ ] Eu sei explicar por que GitOps (pull) reduz a superfície de ataque do pipeline de CD
      comparado a um pipeline tradicional que faz `kubectl apply` de fora do cluster.

## Próxima aula

[Aula 17 — Projeto final e próximos passos ➡](17-projeto-final-proximos-passos.md)
