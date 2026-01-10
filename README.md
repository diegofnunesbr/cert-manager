# cert-manager

Este guia descreve o deployment do **cert-manager** em um cluster Kubernetes, utilizando o manifesto `cert-manager.yaml`.

## Pré-requisitos

- `Kubernetes` instalado
- `kubectl` instalado

---

## Instalar o cert-manager

```bash
git clone https://github.com/diegofnunesbr/cert-manager.git
kubectl apply -f cert-manager/applications/argocd-cert-manager.yaml
```

## Remover o cert-manager

```bash
kubectl delete -f cert-manager/applications/argocd-cert-manager.yaml
kubectl delete namespace cert-manager
```
