# cert-manager

Este guia descreve a instalação e remoção do **cert-manager** em um cluster Kubernetes, utilizando o manifesto `cert-manager.yaml`.

## Pré-requisitos

- `Kubernetes` instalado
- `kubectl` instalado

---

## Instalar o cert-manager

```bash
git clone https://github.com/diegofnunesbr/cert-manager.git
cd cert-manager
kubectl apply -f applications/argocd-cert-manager.yaml
```

## Remover o cert-manager

```bash
cd cert-manager
kubectl delete -f applications/argocd-cert-manager.yaml
kubectl delete namespace cert-manager
```
