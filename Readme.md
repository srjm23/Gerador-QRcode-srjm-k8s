# Gerador de QR Code no Amazon EKS

Manifests Kubernetes para executar o Gerador de QR Code SRJM em um cluster
Amazon EKS.

> O repositório não provisiona o cluster EKS, VPC ou node groups. Ele contém
> apenas os recursos Kubernetes da aplicação.

## Arquitetura

```text
Internet
  └── Load Balancer do NGINX Controller
        └── Ingress ou Gateway API
              └── Service srjm-qrcode:3000
                    └── Deployment srjm-qrcode (3 Pods)
```

O acesso externo utiliza o domínio `geradorqrcode-srjm.uk`. O cert-manager
emite o certificado TLS pelo Let's Encrypt e o armazena no Secret
`geradorqrcode-srjm.uk-tls`.

