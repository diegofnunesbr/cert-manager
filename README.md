# cert-manager

Este guia descreve a instalação e remoção do **cert-manager** em um cluster Kubernetes, utilizando o manifesto `cert-manager.yaml`.

## Pré-requisitos

- `Kubernetes` instalado
- `kubectl` instalado

## Estrutura do repositório

```text
cert-manager/
├── applications/
│   └── argocd.cert-manager.yaml     # Application do Argo CD
├── cert-manager.yaml                # Manifests do cert-manager
└── README.md
```

---

## Gerar o Secret local

```bash
kubectl create secret generic cert-manager-secret -n cert-manager --from-literal=api-token="SEU_TOKEN" --dry-run=client -o yaml > unsealed.secret.yaml
kubeseal --controller-namespace sealed-secrets --format yaml < unsealed.secret.yaml > cert-manager-secret.yaml
rm unsealed.secret.yaml
git add .
git commit -m "first commit"
git push
```

## Instalar o cert-manager

```bash
git clone https://github.com/diegofnunesbr/cert-manager.git
cd cert-manager
kubectl apply -f applications/argocd.cert-manager.yaml
```

## Remover o cert-manager

```bash
cd cert-manager
kubectl delete -f applications/argocd.cert-manager.yaml
kubectl delete namespace cert-manager --ignore-not-found
```
