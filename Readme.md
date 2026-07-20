# Gerador de QR Code no Kubernetes

Manifestos Kubernetes para publicar o **Gerador de QR Code SRJM** com NGINX
Gateway Fabric, entrega contínua via Argo CD e exposição pública pelo Cloudflare.

> Este repositório não provisiona o cluster nem a infraestrutura de rede. Ele
> contém os recursos Kubernetes da aplicação e de sua camada de entrada.

## Arquitetura

```text
Usuário
  │
  ▼
Cloudflare (DNS/proxy e proteção de borda)
  │
  ▼
LoadBalancer do NGINX Gateway Fabric
  │
  ├── geradorqrcode-srjm.uk ──► Service srjm-qrcode:3000
  │                                └── Deployment srjm-qrcode
  │
  └── argocd.geradorqrcode-srjm.uk ──► Service argocd-server:80

Git ──► Argo CD ──► reconciliação dos manifestos no cluster
```

O `Gateway` encerra o TLS dos dois hosts. O cert-manager solicita os
certificados ao Let's Encrypt por meio do desafio HTTP-01. A aplicação é
executada no namespace `qrcode-system`, com duas réplicas iniciais e atualização
do tipo `RollingUpdate`.

## Recursos

| Manifesto | Função |
| --- | --- |
| `deployment.yml` | Executa a imagem da aplicação na porta `3000`, com probes e limites de recursos |
| `service.yml` | Expõe os Pods internamente por um Service `ClusterIP` |
| `hpa.yml` | Escala de 2 a 5 réplicas quando CPU ou memória atingem 70% |
| `gateway.yml` | Define os listeners HTTP/HTTPS da aplicação e do Argo CD |
| `http-route.yml` | Encaminha o domínio principal para a aplicação |
| `argocd-route/httproute-argocd.yml` | Encaminha o subdomínio do Argo CD para `argocd-server` |
| `production_issuer-Gateway.yml` | Configura o emissor de produção do Let's Encrypt |
