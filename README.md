# Cert-Manager

Este guia descreve a instalação do **cert-manager** em um cluster Kubernetes usando o manifesto `cert-manager.yaml`, que já contém as permissões RBAC, o webhook e o `ClusterIssuer` **selfsigned** necessário para provisionar certificados TLS.

## Pré-requisitos

- `Kubernetes` instalado

---

## Instalação

1. **Aplique o manifesto no cluster:**

```bash
kubectl apply -f cert-manager.yaml
