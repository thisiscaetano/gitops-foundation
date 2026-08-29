# br-foundation-gitops

Imagem Docker padrão para deploy GitOps via ArgoCD + Kustomize na Wevy.

Ao ser executada, a imagem clona o repositório de manifests (`forfun-global-manifest`), atualiza a imagem da aplicação no overlay do Kustomize e faz push na branch `main`. O ArgoCD detecta a mudança e aplica o deploy automaticamente.

---

## Como funciona

```
CI Pipeline
    │
    ▼
Build & push da imagem da aplicação → ACR
    │
    ▼
Executa br-foundation-gitops
    │
    ▼
Clona forfun-global-manifest
Atualiza apps/<APP_NAME>/overlays/<OVERLAY>/kustomization.yaml
Push → main
    │
    ▼
ArgoCD detecta mudança e faz o deploy
```

---

## Variáveis de ambiente

| Variável | Obrigatório | Descrição |
|---|---|---|
| `APP_NAME` | ✅ | Nome da aplicação. Deve corresponder ao diretório em `apps/` no manifest repo |
| `TAG` | ✅ | Tag da imagem a ser deployada (ex: short SHA do commit) |
| `ARTIFACT_REGISTRY` | ✅ | Registry + nome da imagem (ex: `wevyacrfoundation.azurecr.io/minha-app`) |
| `GLOBAL_MANIFEST_REPOSITORY` | ✅ | URL SSH do repositório de manifests (ex: `git@github.com:Wevy-SRE/forfun-global-manifest.git`) |
| `GLOBAL_MANIFEST_REPOSITORY_DOMAIN` | ✅ | Domínio do repositório (ex: `github.com`) |
| `GLOBAL_MANIFEST_SSH_PRIVATE_KEY` | ✅ | Chave SSH privada em **base64** com permissão de **escrita** no manifest repo |
| `GLOBAL_MANIFEST_OVERLAY` | ✅ | Nome do overlay Kustomize (ex: `production`, `staging`) |
| `VERSION` | ❌ | Versão para o `datadog-version.yaml`. Se omitido, usa `YYYY.MM.DD.HH.MM` |

> **Atenção:** `GLOBAL_MANIFEST_SSH_PRIVATE_KEY` deve ser a chave privada codificada em base64:
> ```bash
> cat ~/.ssh/id_rsa | base64 -w 0
> ```

---

## Pré-requisitos no repositório de manifests

Cada aplicação precisa seguir a estrutura abaixo no `forfun-global-manifest`:

```
apps/
└── <APP_NAME>/
    └── overlays/
        └── <OVERLAY>/
            ├── kustomization.yaml   ← deve ter a seção images com name: image-set
            └── deployment.yaml      ← container deve ter image: image-set
```

### kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  # ... outros recursos

images:
  - name: image-set
    newName: ""
    newTag: ""
```

### deployment.yaml

```yaml
containers:
  - name: minha-app
    image: image-set   # placeholder substituído pelo Kustomize
```

---

## Uso no GitHub Actions

```yaml
- name: Deploy
  env:
    GLOBAL_MANIFEST_SSH_PRIVATE_KEY: ${{ secrets.GLOBAL_MANIFEST_SSH_PRIVATE_KEY }}
  run: |
    ARTIFACT_REGISTRY="wevyacrfoundation.azurecr.io/${{ env.IMAGE_NAME }}"

    docker run --rm \
      -e APP_NAME=${{ env.IMAGE_NAME }} \
      -e TAG=${GITHUB_SHA::7} \
      -e VERSION=${GITHUB_SHA::7} \
      -e GLOBAL_MANIFEST_OVERLAY=production \
      -e GLOBAL_MANIFEST_REPOSITORY=git@github.com:Wevy-SRE/forfun-global-manifest.git \
      -e GLOBAL_MANIFEST_REPOSITORY_DOMAIN=github.com \
      -e ARTIFACT_REGISTRY="$ARTIFACT_REGISTRY" \
      -e GLOBAL_MANIFEST_SSH_PRIVATE_KEY="$GLOBAL_MANIFEST_SSH_PRIVATE_KEY" \
      wevyacrfoundation.azurecr.io/br-foundation-gitops:1.1
```

### Secrets necessários no repositório

| Secret | Descrição |
|---|---|
| `GLOBAL_MANIFEST_SSH_PRIVATE_KEY` | Chave SSH privada em base64 com acesso de **escrita** ao `forfun-global-manifest` |

---

## Build e publicação da imagem

```bash
# A partir do diretório br-foundation-gitops/
docker build -f devops/Dockerfile -t wevyacrfoundation.azurecr.io/br-foundation-gitops:1.1 .
docker push wevyacrfoundation.azurecr.io/br-foundation-gitops:1.1
```

> Ao atualizar o `entry.sh` ou o `Dockerfile`, incremente a tag da imagem e atualize todas as pipelines que a referenciam.

---

## Chave SSH — configuração no GitHub

A chave SSH precisa ter acesso de **escrita** ao repositório `forfun-global-manifest`:

1. Gere um par de chaves:
   ```bash
   ssh-keygen -t ed25519 -C "gitops-bot" -f ~/.ssh/gitops_key -N ""
   ```
2. Adicione a chave pública em `forfun-global-manifest` → **Settings → Deploy keys → Add deploy key**
   - Marque **"Allow write access"**
3. Encode a chave privada em base64:
   ```bash
   cat ~/.ssh/gitops_key | base64 -w 0
   ```
4. Salve o resultado como secret `GLOBAL_MANIFEST_SSH_PRIVATE_KEY` no repositório da aplicação no GitHub
