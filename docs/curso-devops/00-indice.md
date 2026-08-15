# Curso de DevOps na Prática — usando o Online Boutique GitOps Demo

Bem-vindo(a)! Este curso usa **este repositório** como laboratório único e progressivo para
ensinar DevOps do zero até um cenário "produção-like" na AWS: containers, Kubernetes,
Infraestrutura como Código, GitOps, CI/CD, observabilidade, service mesh, autoscaling e
segurança.

> Diferente de um curso teórico, aqui **cada aula aponta para arquivos reais deste repositório**.
> Você vai ler o código/manifesto, entender o "porquê" por trás dele e, quando fizer sentido,
> executá-lo.

## Para quem é este curso

Para quem já sabe programar (não precisa ser especialista) e quer entender como uma aplicação
vai do código-fonte até rodar de forma confiável em produção, com prática de mercado real:
IaC, Kubernetes, GitOps, CI/CD, observabilidade e segurança.

## Como o curso está organizado

O curso segue a **ordem de construção da plataforma**, do mais simples (conceitos e containers)
até o mais avançado (service mesh, segurança, projeto final). Cada aula tem:

- **Objetivos** — o que você vai saber fazer ao final.
- **Conceito** — a teoria de DevOps por trás do assunto.
- **Na prática, neste repositório** — os arquivos reais que implementam o conceito.
- **Mão na massa** — comandos para executar/validar.
- **Checklist** — perguntas para você confirmar o aprendizado.
- **Próxima aula** — link para continuar.

## Sumário

| # | Aula | Tema |
|---|------|------|
| 01 | [Fundamentos de DevOps e GitOps](01-fundamentos-devops-e-gitops.md) | O que é DevOps, CI/CD, IaC e GitOps. Visão geral da arquitetura da plataforma. |
| 02 | [Arquitetura da aplicação](02-arquitetura-da-aplicacao.md) | Microsserviços do Online Boutique, comunicação gRPC, persistência de dados. |
| 03 | [Containers e Docker](03-containers-e-docker.md) | Imagens, Dockerfiles dos serviços, build/tag/push para o GHCR. |
| 04 | [Fundamentos de Kubernetes](04-fundamentos-kubernetes.md) | Pods, Deployments, Services, manifests puros do projeto. |
| 05 | [Terraform: Infraestrutura como Código](05-terraform-infraestrutura-como-codigo.md) | VPC, cluster EKS, bastion host, backend remoto no S3. |
| 06 | [Acesso ao cluster, Load Balancer e Gateway API](06-acesso-cluster-loadbalancer-gateway-api.md) | kubectl no EKS, AWS Load Balancer Controller, Gateway API, External DNS. |
| 07 | [Kustomize: variações de deploy](07-kustomize-variacoes-de-deploy.md) | Base + components, customização sem duplicação. |
| 08 | [Helm: empacotando a aplicação](08-helm-empacotando-a-aplicacao.md) | Chart, values, templates, publicação OCI no GHCR. |
| 09 | [GitOps com ArgoCD](09-gitops-com-argocd.md) | Instalação, Application CR, sync automático, self-heal. |
| 10 | [CI com GitHub Actions](10-ci-com-github-actions.md) | Build, scan de vulnerabilidades (Trivy), push de imagens. |
| 11 | [CD: fechando o loop com Argo CD Image Updater](11-cd-argocd-image-updater.md) | Do commit à imagem rodando em produção, sem `kubectl apply` manual. |
| 12 | [Observabilidade: métricas](12-observabilidade-metricas-prometheus-grafana.md) | Prometheus, Grafana, Alertmanager + Slack. |
| 13 | [Observabilidade: logs](13-observabilidade-logs-eck-kibana.md) | Elastic (ECK), Filebeat, Kibana. |
| 14 | [Escalabilidade: HPA](14-escalabilidade-hpa.md) | metrics-server, Horizontal Pod Autoscaler, GitOps drift. |
| 15 | [Service Mesh com Istio](15-service-mesh-istio.md) | VirtualService, Gateway, ServiceEntry. |
| 16 | [Segurança e boas práticas](16-seguranca-boas-praticas.md) | Revisão OWASP-style do projeto: IAM, secrets, rede, scanning. |
| 17 | [Projeto final e próximos passos](17-projeto-final-proximos-passos.md) | Capstone, cleanup e roadmap para ir além. |

## Pré-requisitos

- Conhecimento básico de Linux/terminal.
- Git básico (clone, commit, push, branch).
- Não é necessário saber Kubernetes ou Terraform antes — o curso ensina do zero.
- Para as aulas práticas com AWS (05 em diante): uma conta AWS (a infraestrutura gera custo
  real — sempre destrua os recursos ao final, ver aula 17).

## Convenções usadas no curso

- Caminhos de arquivo são sempre relativos à raiz do repositório, ex.: `terraform/eks.tf`.
- Blocos `bash` são comandos para rodar no terminal.
- `[!IMPORTANTE]` marca decisões de arquitetura que valem a pena entender bem, não só copiar.

Vamos começar pela [Aula 01 — Fundamentos de DevOps e GitOps](01-fundamentos-devops-e-gitops.md).
