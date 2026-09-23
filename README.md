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
├── mimir-agents-ca.yaml             # CA interna do mTLS entre o Alloy das VMs e o Mimir
└── README.md
```

## CA interna do mTLS do Mimir (`mimir-agents-ca.yaml`)

Além do `letsencrypt-clusterissuer` (certificados públicos dos Ingress),
este repositório cria uma CA própria, só pra autenticar o Alloy das VMs
no envio de métricas pro Mimir (mesmo modelo da empresa, onde os agentes
enviam pro gateway `telemetry-agents` com certificado de cliente):

- `ClusterIssuer selfsigned` assina o certificado da CA
  (`Certificate mimir-agents-ca`, 1 ano, Secret `mimir-agents-ca` na
  namespace `cert-manager`).
- `ClusterIssuer mimir-agents-ca` usa essa CA pra emitir os certificados
  de cliente (hoje só o `alloy-mtls-client`, no repositório `rundeck`).
- O Ingress do Mimir confia nela via `auth-tls-secret: cert-manager/mimir-agents-ca`.

A CA é renovada sozinha todo ano (aos 2/3 da validade, o padrão do
cert-manager) **mantendo a mesma chave privada**, por causa do
`rotationPolicy: Never`. Por isso a renovação é invisível: os certificados
de cliente já emitidos continuam sendo aceitos. **Não tire essa linha:** a
partir do cert-manager 1.18 o padrão virou `Always`, e sem ela cada
renovação anual geraria uma chave nova, invalidando o certificado de todas
as VMs até alguém rodar o job `install-alloy` em cada uma.

A chave da CA é gerada pelo próprio cert-manager dentro do cluster e não
vai pro git. Num cluster recriado do zero nasce uma CA nova: o cert-manager
reemite o certificado de cliente sozinho, e basta rodar o job
`install-alloy` do Rundeck de novo em cada VM pra ela receber o novo.

## Gerar o SealedSecret cert-manager-secret

```bash
read -rsp "Token Cloudflare: " T; echo
kubectl create secret generic cert-manager-secret -n cert-manager \
  --from-file=api-token=<(printf '%s' "$T") \
  --dry-run=client -o yaml \
  | kubeseal --controller-name sealed-secrets --controller-namespace kube-system \
      --format yaml > sealed.secret.yaml
unset T
```

O `read -s` pede o token sem mostrar na tela, e ele nunca aparece na
linha de comando: nada vai pro `~/.bash_history` e nenhum arquivo não
selado é gravado em disco.

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
