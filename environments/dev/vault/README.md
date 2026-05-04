# Vault Setup for Dev Environment

This directory contains manifests and documentation for HashiCorp Vault setup in the dev EKS cluster.

## Architecture

- **Vault Server**: Runs in `vault` namespace as a 3-replica HA Raft cluster
- **Auto-Unseal**: AWS KMS key (`alias/zt-dev-platform-eks-vault-unseal`)
- **Auth Method**: Kubernetes auth for workload identity
- **Secret Injection**: Vault Agent sidecar injector
- **UI Access**: https://vault.test.zerotheft.com (after initialization)

## Automated Setup via GitHub Actions

### Step 1: Initialize Vault

Run the **Vault Setup** workflow with action `init`:

```
GitHub Actions → Vault Setup → Run workflow → action: init
```

This will:
1. Run `vault operator init` on the active Vault pod
2. Create Kubernetes secret `vault-root-token` in the vault namespace
3. Store the root token in AWS Secrets Manager: `zt-dev-vault-root-token`
4. Display the root token in the workflow output (save it securely!)

With KMS auto-unseal, Vault will automatically unseal after initialization.

### Step 2: Configure Vault

Run the **Vault Setup** workflow with action `configure`:

```
GitHub Actions → Vault Setup → Run workflow → action: configure
```

This will:
1. Enable Kubernetes auth method
2. Enable KV v2 secrets engine at `secret/`
3. Create sample secrets for all services
4. Create Vault policies for each service
5. Create Kubernetes auth roles mapping service accounts to policies

### Step 3: Verify

Run the **Vault Setup** workflow with action `status`:

```
GitHub Actions → Vault Setup → Run workflow → action: status
```

## Manual Verification (if needed)

If you have kubectl access configured:

```bash
# Check Vault pods
kubectl get pods -n vault

# Check Vault status
kubectl exec -n vault vault-0 -- sh -c 'VAULT_ADDR=http://127.0.0.1:8200 vault status'

# View Vault logs
kubectl logs -n vault vault-0
```

## Service Secret Injection

Each service has Vault Agent sidecar injection configured via Kustomize patches:

| Service | Vault Role | Secrets Path |
|---------|-----------|--------------|
| auth-service | auth-service | `secret/auth-service/db`, `secret/auth-service/jwt` |
| audit-service | audit-service | `secret/audit-service/db` |
| billing-service | billing-service | `secret/billing-service/db`, `secret/billing-service/stripe` |
| notification-service | notification-service | `secret/notification-service/smtp`, `secret/notification-service/slack` |
| saas-app | saas-app | `secret/saas-app/*`, `secret/shared/*` |

### How It Works

1. The deployment has Vault Agent annotations
2. Vault Agent injector (running in cluster) watches for these annotations
3. When pod is created, injector adds a Vault Agent sidecar
4. Sidecar authenticates using the pod's service account JWT
5. Sidecar fetches secrets and writes them to `/vault/secrets/` files
6. Main application container reads secrets from these files

### Application Integration

Your application should read secrets from the injected files:

```bash
# Secrets are written as environment variable exports
source /vault/secrets/db
source /vault/secrets/jwt

# Or read as key=value pairs
DB_HOST=$(grep DB_HOST /vault/secrets/db | cut -d= -f2)
```

## Adding New Secrets

To add new secrets for a service:

1. Add the secret to Vault (via workflow or kubectl exec):
   ```bash
   kubectl exec -n vault vault-0 -- sh -c 'VAULT_ADDR=http://127.0.0.1:8200 VAULT_TOKEN=<token> vault kv put secret/<service>/<name> key=value'
   ```

2. Update the service's `vault-injection.yaml` patch with new annotations

3. Commit and push — ArgoCD will sync the changes

## Troubleshooting

### Vault not initializing
- Check KMS key permissions: IRSA role must have `kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey`
- Check Vault pod logs: `kubectl logs -n vault vault-0`

### Sidecar not injected
- Verify Vault Agent injector is running: `kubectl get pods -n vault | grep injector`
- Check deployment has correct annotations
- Verify service account exists and matches Vault role

### Authentication failures
- Check Kubernetes auth config: `vault read auth/kubernetes/config`
- Verify service account token is valid
- Check Vault role binding: `vault read auth/kubernetes/role/<role-name>`

## Security Notes

- Root token is stored in AWS Secrets Manager AND as a Kubernetes secret
- For production, use short-lived tokens and enable audit logging
- Rotate secrets regularly using Vault's dynamic secret engines
- Consider enabling TLS for Vault UI and API in production
