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

Intalando a versão do gateway-nginx

kubectl kustomize "https://github.com/nginxinc/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.1" | kubectl apply -f -
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.6.1/deploy/crds.yaml

kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.6.1/deploy/default/deploy.yaml

https://github.com/nginx/nginx-gateway-fabric

habilitando hpa

kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml