# whatsapp-bridge Helm Chart

Roda o [Baileys](https://github.com/WhiskeySockets/Baileys) + servidor MCP HTTP no k3s. Permite que o Claude Code leia e envie mensagens WhatsApp de um número dedicado via ferramentas MCP.

---

## Arquitetura

```
Claude Code (host)
    │  http://localhost:8765/mcp
    │  Authorization: Bearer <token>
    ▼
[systemd port-forward] ─── kubectl port-forward -n whatsapp svc/whatsapp-bridge 8765:3000
    │
    ▼
[Pod: whatsapp-bridge]  node:20-alpine
    ├── bridge.ts          ← Baileys + MCP HTTP server (porta 3000)
    ├── /data/auth/        ← Auth state WhatsApp (PVC)
    └── /data/db.sqlite    ← Mensagens + chats + contatos (PVC)
```

---

## Pré-requisitos no cluster

- k3s com `local-path` StorageClass.
- [Sealed Secrets controller](https://github.com/bitnami-labs/sealed-secrets) instalado.
- ArgoCD instalado (namespace `argocd`).

---

## Deploy

O chart é sincronizado automaticamente pelo ArgoCD via `charts/root-app/templates/whatsapp-bridge.yaml`.

Para deploy manual (teste):

```bash
helm template /caminho/para/charts/whatsapp-bridge --debug
helm upgrade --install whatsapp-bridge /caminho/para/charts/whatsapp-bridge -n whatsapp --create-namespace
```

---

## Configuração do SealedSecret (T4)

Gerar token e cifrar **uma única vez**:

```bash
# 1. Gerar token
TOKEN=$(openssl rand -base64 32)

# 2. Salvar localmente (nunca commitar)
mkdir -p ~/.config/whatsapp-bridge
 echo "$TOKEN" > ~/.config/whatsapp-bridge/token
chmod 600 ~/.config/whatsapp-bridge/token

# 3. Verificar que o Sealed Secrets CRD existe
kubectl get crd sealedsecrets.bitnami.com

# 4. Gerar e substituir o sealed secret no chart
kubectl create secret generic whatsapp-bridge-secrets \
  --namespace whatsapp \
  --from-literal=mcp-bearer-token="$TOKEN" \
  --dry-run=client -o yaml \
| kubeseal --scope namespace-wide --namespace whatsapp -o yaml \
> charts/whatsapp-bridge/templates/sealed-secret.yaml

# 5. Commit + push
git add charts/whatsapp-bridge/templates/sealed-secret.yaml
git commit -m "feat: add whatsapp-bridge sealed secret"
git push
```

> ⚠️ O arquivo `sealed-secret.yaml` no repositório contém apenas o valor **cifrado** — é seguro commitar em repositório público.

---

## Pareamento QR (T8)

Ver: `pessoal/whatsapp/PAREAMENTO.md`

---

## Port-forward systemd (T9)

```bash
# Habilitar e iniciar
systemctl --user enable --now whatsapp-portforward.service

# Verificar status
systemctl --user status whatsapp-portforward

# Habilitar ao boot mesmo sem login (lingering)
loginctl enable-linger julio
```

---

## SQLite Schema

| Tabela | Descrição |
|---|---|
| `chats` | Chats conhecidos, indexados por `jid`, com `last_message_ts` |
| `contacts` | Contatos sincronizados do WhatsApp |
| `messages` | Todas as mensagens (texto + metadado de mídia). `has_media=1` quando não é texto. |

Colunas relevantes de `messages`:

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | TEXT | ID único da mensagem (Baileys) |
| `chat_jid` | TEXT | JID do chat |
| `from_jid` | TEXT | `'me'` ou JID do remetente |
| `body` | TEXT | Texto da mensagem (null se apenas mídia sem legenda) |
| `timestamp` | INTEGER | Unix timestamp |
| `from_me` | INTEGER | 1 = enviada pelo bridge, 0 = recebida |
| `has_media` | INTEGER | 1 = mensagem tem mídia (não baixada) |
| `media_type` | TEXT | `'image'`, `'video'`, `'audio'`, `'document'` ou null |

---

## Apagar dados manualmente

```bash
# 1. Deletar PVC (apaga auth state + todas as mensagens)
kubectl delete pvc whatsapp-bridge-data -n whatsapp

# 2. Reiniciar pod (vai pedir novo QR)
kubectl rollout restart deploy/whatsapp-bridge -n whatsapp
```

> ⚠️ Após deletar o PVC, um novo pareamento QR é necessário.

---

## Manutenção

### Atualizar Baileys

Editar `package.json` dentro do `configmap-code.yaml`, incrementar a versão de `@whiskeysockets/baileys`, fazer commit e push. O ArgoCD recria o pod automaticamente.

### Backup do SQLite

```bash
kubectl cp whatsapp/$(kubectl get pod -n whatsapp -l app=whatsapp-bridge -o jsonpath='{.items[0].metadata.name}'):/data/db.sqlite ./db-backup-$(date +%Y%m%d).sqlite
```

---

## Variáveis de ambiente do pod

| Var | Valor | Origem |
|---|---|---|
| `DATA_DIR` | `/data` | hardcoded no Deployment |
| `MCP_PORT` | `3000` | `values.yaml` |
| `MCP_BEARER_TOKEN` | `<cifrado>` | SealedSecret |
| `LOG_LEVEL` | `info` | hardcoded no Deployment |
| `NODE_ENV` | `production` | hardcoded no Deployment |
