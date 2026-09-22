# cert-manager

Este guia descreve a instalação e remoção do **cert-manager** em um
cluster Kubernetes, utilizando o manifesto `cert-manager.yaml` (manifest
oficial vendorizado + um `SealedSecret` e um `ClusterIssuer` próprios,
no final do arquivo).

O `ClusterIssuer` (`letsencrypt-clusterissuer`) emite certificados reais
da Let's Encrypt via desafio **DNS-01 no Cloudflare** - é o mecanismo
certo pra domínios que resolvem pra IP privado (não dá pra usar HTTP-01,
a Let's Encrypt nunca alcançaria o IP de fora), que é o caso de todo
serviço desse homelab.

## Pré-requisitos

- `Kubernetes` instalado
- `kubectl` e `kubeseal` instalados
- ArgoCD instalado, com `Sealed Secrets` já saudável (via `core-config`
  do repositório `argocd`)
- Zona `diegofnunesbr.com` já criada no Cloudflare, com um API Token
  com escopo `DNS Write` (ver repositório `dns`)

## Estrutura do repositório

```text
cert-manager/
├── applications/
│   └── argocd.cert-manager.yaml     # Application do Argo CD
├── cert-manager.yaml                # Manifests do cert-manager + SealedSecret + ClusterIssuer
└── README.md
```

## Gerar o SealedSecret cert-manager-secret

```bash
kubectl create secret generic cert-manager-secret -n cert-manager \
  --from-literal=api-token="SEU_TOKEN_DO_CLOUDFLARE" \
  --dry-run=client -o yaml > unsealed.secret.yaml

kubeseal --controller-name sealed-secrets --controller-namespace kube-system \
  --format yaml < unsealed.secret.yaml > sealed.secret.yaml

rm -f unsealed.secret.yaml
```

**Atenção pro nome/namespace do controller**: mudou depois que o
`sealed-secrets` passou a ser gerenciado pelo ArgoCD (via `core-config`)
- hoje é `sealed-secrets` na namespace `kube-system`, não mais
`sealed-secrets-controller` na namespace `sealed-secrets` (esse era o
padrão antigo, de antes da migração). Confira sempre com:

```bash
kubectl -n kube-system get svc -l app.kubernetes.io/name=sealed-secrets
```

Copie o valor de `spec.encryptedData.api-token` do `sealed.secret.yaml`
gerado e substitua o bloco `encryptedData.api-token` do `SealedSecret`
já existente no final do `cert-manager.yaml`, depois commite e dê push -
a Application lê do GitHub, não do seu clone local.

**Se o `ClusterIssuer`/app ficar `Degraded` com `no key could decrypt`**:
o `SealedSecret` no repositório foi criptografado com uma chave de
sealed-secrets que não existe mais no cluster (acontece se o cluster foi
recriado, ou o sealed-secrets foi reinstalado do zero em algum momento).
A única solução é gerar um `SealedSecret` novo (passos acima, com a
chave *atual*) e atualizar o `cert-manager.yaml` - não tem como
recuperar uma chave perdida.

## Instalar o cert-manager

```bash
git clone https://github.com/diegofnunesbr/cert-manager.git
cd cert-manager
kubectl apply -f applications/argocd.cert-manager.yaml
```

## Verificar

```bash
kubectl -n cert-manager get pods
kubectl get clusterissuer letsencrypt-clusterissuer -o jsonpath='{.status.conditions}'
```

Deve mostrar os 3 pods (`cert-manager`, `cert-manager-cainjector`,
`cert-manager-webhook`) `Running`, e o `ClusterIssuer` com
`"reason":"ACMEAccountRegistered","status":"True"`. Isso confirma que a
conta ACME registrou - a emissão de certificado de verdade só é testada
quando algum serviço (Ingress com a anotação
`cert-manager.io/cluster-issuer: letsencrypt-clusterissuer`) pedir um.

## Remover o cert-manager

```bash
cd cert-manager
kubectl delete -f applications/argocd.cert-manager.yaml
kubectl delete namespace cert-manager --ignore-not-found
```
