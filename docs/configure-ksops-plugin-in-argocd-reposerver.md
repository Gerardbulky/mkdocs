# KSOPS and Slack Notifications Setup for ArgoCD

This document outlines the implementation steps for configuring KSOPS (Kustomize Secret OPerationS) with ArgoCD and setting up Slack notifications for application status changes.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [KSOPS Setup](#ksops-setup)
3. [Slack Notifications Setup](#slack-notifications-setup)
4. [Testing](#testing)
5. [Troubleshooting](#troubleshooting)

## Prerequisites

- ArgoCD installed and running
- kubectl access to the cluster
- Slack workspace with webhook/bot token permissions
- Age encryption tool (for SOPS)

## KSOPS Setup

### 1. Install SOPS
Visit the latest release page:
https://github.com/getsops/sops/releases/latest
```bash
# Download and install SOPS
curl -LO https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64
chmod +x sops-v3.8.1.linux.amd64
sudo mv sops-v3.8.1.linux.amd64 /usr/local/bin/sops


# Verify installation
sops --version
```

### 2. Install Age Encryption Tool

```bash
# Install age
sudo apt update && sudo apt install age -y

# Create directory for age keys
mkdir -p ~/.config/sops/age

# Generate age key
age-keygen -o ~/.config/sops/age/keys.txt

# Show your public key (save this)
cat ~/.config/sops/age/keys.txt | grep "# public key:"
```

### 3. Create SOPS Configuration

Create `.sops.yaml` in your ArgoCD directory:

```yaml
creation_rules:
  - path_regex: \.enc\.yaml$
    age: age1edd6a5h864964c060u0g4chtghrngsys7rcwheny4gwrszmm34lqpru0jk
```

### 4. Create KSOPS Plugin ConfigMap

Create `ksops-plugin-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ksops-plugin-config
  namespace: argocd
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: ksops
    spec:
      version: v1.0
      init:
        command: ["/bin/sh", "-c"]
        args: ["kustomize build --enable-alpha-plugins ."]
      generate:
        command: ["/bin/sh", "-c"]
        args: ["kustomize build --enable-alpha-plugins . | sops --decrypt /dev/stdin"]
      discover:
        find:
          command: ["/bin/sh", "-c"]
          args: ["find . -name '*.enc.yaml' -o -name '*.enc.yml' -o -name 'kustomization.yaml' -o -name 'kustomization.yml' | head -1"]
```

### 4b. Apply KSOPS Plugin

```bash
kubectl apply -f ksops-plugin-configmap.yaml
```

### 5. Configure ksops plugin in ArgoCD repoServer
![Configure ksops plugin in ArgoCD repoServer](img/configure-ksops-plugin-in-argocd-reposerver.png)

Add the following configuration to your ArgoCD repoServer deployment. The configuration includes:

  - Volume mounts for kustomize and sops binaries
  - Mount for the plugin configuration file
  - KSOPS sidecar container with security context
  - Proper command and args for the sidecar:

```yaml
  - mountPath: /usr/local/bin/kustomize
    name: custom-tools
    subPath: kustomize
  - mountPath: /usr/local/bin/sops
    name: custom-tools
    subPath: sops
  - mountPath: /home/argocd/cmp-server/config/plugin.yaml
    name: ksops-plugin-config
    subPath: plugin.yaml
  - name: ksops
    image: viaductoss/ksops:v4.3.2
    command: ["/bin/sh", "-c"]
    args:
      - echo "Starting ksops sidecar" && cp /usr/local/bin/kustomize /custom-tools/ && cp /usr/local/bin/sops /custom-tools/ && while true; do sleep 30; done
    volumeMounts:
      - mountPath: /custom-tools
        name: custom-tools
      - mountPath: /tmp
        name: tmp
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      seccompProfile:
        type: RuntimeDefault
```
### 5b. Add the ksops-plugin-config volume
![Configure ksops plugin in ArgoCD repoServer](img/ksops-plugin-config.png)

The configuration includes:

  - var-files volume with emptyDir
  - plugins volume with emptyDir
  - custom-tools volume with emptyDir
  - ksops-plugin-config volume using the configMap
  
Add the following volumes configuration:

```yaml
    volumes:
      - name: var-files
        emptyDir: {}
      - name: plugins
        emptyDir: {}
      - name: custom-tools
        emptyDir: {}
      - configMap:
          name: ksops-plugin-config
        name: ksops-plugin-config
```

```bash
# Apply the updated ArgoCD configuration
kubectl rollout restart deployment/argocd-repo-server -n argocd
```

### 6. Update .gitignore

Add unencrypted secret files to `.gitignore`:

```
# Unencrypted secret files
argocd-notifications-secret.yaml
*-secret.yaml
```

## Slack Notifications Setup

### 1. Create Slack Webhook/Bot Token

1. Go to your Slack workspace settings
2. Create a new app or bot
3. Get the webhook URL and/or bot token
4. Note the channel where notifications should be sent

### 2. Create Encrypted Secret

```bash
# Create unencrypted secret file
cat > argocd-notifications-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  slack-token: xoxb-your-actual-slack-bot-token-here
  slack-webhook-url: https://hooks.slack.com/services/your/actual/webhook/url
EOF

# Encrypt with SOPS
sops -e argocd-notifications-secret.yaml > argocd-notifications-secret.enc.yaml

# Apply the encrypted version
sops -d argocd-notifications-secret.enc.yaml | kubectl apply -f -

# Remove unencrypted file
rm argocd-notifications-secret.yaml

# Check if the file is encrypted/ cd & run command in argocd directory
head -5 argocd-notifications-secret.yaml
```

### 3. Create Notifications ConfigMap

Create `argocd-notifications-configmap.yaml` with the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
    apiURL: https://slack.com/api
  service.webhook.slack-webhook: |
    url: $slack-webhook-url
    headers:
    - name: Content-Type
      value: application/json
  
  # Problem notification templates
  template.app-health-degraded: |
    webhook:
      slack-webhook:
        method: POST
        body: |
          {
            "channel": "#dev-argo-alerts",
            "username": "ArgoCD Alert",
            "icon_emoji": ":warning:",
            "attachments": [{
              "color": "danger",
              "title": "🚨 Application Health Alert",
              "fields": [
                {
                  "title": "Application",
                  "value": "{{.app.metadata.name}}",
                  "short": true
                },
                {
                  "title": "Health Status",
                  "value": "{{.app.status.health.status}}",
                  "short": true
                },
                {
                  "title": "Environment",
                  "value": "Development",
                  "short": true
                },
                {
                  "title": "Message",
                  "value": "{{if .app.status.health.message}}{{.app.status.health.message}}{{else}}Application health is degraded{{end}}",
                  "short": false
                }
              ],
              "footer": "MediaFlow ArgoCD Alert System"
            }]
          }
  
  template.app-sync-failed: |
    webhook:
      slack-webhook:
        method: POST
        body: |
          {
            "channel": "#dev-argo-alerts",
            "username": "ArgoCD Alert",
            "icon_emoji": ":x:",
            "attachments": [{
              "color": "danger",
              "title": "❌ Application Sync Failed",
              "fields": [
                {
                  "title": "Application",
                  "value": "{{.app.metadata.name}}",
                  "short": true
                },
                {
                  "title": "Sync Status",
                  "value": "{{.app.status.sync.status}}",
                  "short": true
                },
                {
                  "title": "Environment",
                  "value": "Development",
                  "short": true
                },
                {
                  "title": "Error",
                  "value": "{{if .app.status.operationState.message}}{{.app.status.operationState.message}}{{else}}Application is out of sync{{end}}",
                  "short": false
                }
              ],
              "footer": "MediaFlow ArgoCD Alert System"
            }]
          }
  
  # # Resolution notification templates
  # template.app-health-recovered: |
  #   webhook:
  #     slack-webhook:
  #       method: POST
  #       body: |
  #         {
  #           "channel": "#dev-argo-alerts",
  #           "username": "ArgoCD Alert",
  #           "icon_emoji": ":white_check_mark:",
  #           "attachments": [{
  #             "color": "good",
  #             "title": "✅ Application Health Recovered",
  #             "fields": [
  #               {
  #                 "title": "Application",
  #                 "value": "{{.app.metadata.name}}",
  #                 "short": true
  #               },
  #               {
  #                 "title": "Health Status",
  #                 "value": "{{.app.status.health.status}}",
  #                 "short": true
  #               },
  #               {
  #                 "title": "Environment",
  #                 "value": "Development",
  #                 "short": true
  #               },
  #               {
  #                 "title": "Message",
  #                 "value": "Application health has been restored",
  #                 "short": false
  #               }
  #             ],
  #             "footer": "MediaFlow ArgoCD Alert System"
  #           }]
  #         }
  
  # template.app-sync-succeeded: |
  #   webhook:
  #     slack-webhook:
  #       method: POST
  #       body: |
  #         {
  #           "channel": "#dev-argo-alerts",
  #           "username": "ArgoCD Alert",
  #           "icon_emoji": ":heavy_check_mark:",
  #           "attachments": [{
  #             "color": "good",
  #             "title": "✅ Application Sync Succeeded",
  #             "fields": [
  #               {
  #                 "title": "Application",
  #                 "value": "{{.app.metadata.name}}",
  #                 "short": true
  #               },
  #               {
  #                 "title": "Sync Status",
  #                 "value": "{{.app.status.sync.status}}",
  #                 "short": true
  #               },
  #               {
  #                 "title": "Environment",
  #                 "value": "Development",
  #                 "short": true
  #               },
  #               {
  #                 "title": "Message",
  #                 "value": "Application successfully synchronized",
  #                 "short": false
  #               }
  #             ],
  #             "footer": "MediaFlow ArgoCD Alert System"
  #           }]
  #         }
  
  # Triggers
  trigger.on-health-degraded: |
    - description: Application has degraded health
      send: [app-health-degraded]
      when: app.status.health.status == 'Degraded'

  trigger.on-sync-out-of-sync: |
    - description: Application is out of sync
      send: [app-sync-failed]
      when: app.status.sync.status == 'OutOfSync'
  
  trigger.on-sync-failed: |
    - description: Application sync failed
      send: [app-sync-failed]
      when: app.status.sync.status == 'OutOfSync' and app.status.operationState.phase == 'Failed'
  
  trigger.on-sync-status-unknown: |
    - description: Application sync status is unknown
      send: [app-health-degraded]
      when: app.status.sync.status == 'Unknown'
  
  # trigger.on-health-recovered: |
  #   - description: Application health has recovered
  #     send: [app-health-recovered]
  #     when: app.status.health.status == 'Healthy'
  
  # trigger.on-sync-succeeded: |
  #   - description: Application sync succeeded
  #     send: [app-sync-succeeded]
  #     when: app.status.sync.status == 'Synced' and app.status.operationState.phase == 'Succeeded'
  
  subscriptions: |
    - recipients:
      - slack-webhook
      triggers:
      - on-health-degraded
      - on-sync-failed
      - on-sync-status-unknown
      - on-sync-out-of-sync
      # - on-health-recovered
      # - on-sync-succeeded
```

### 4. Apply Configurations

```bash
# Apply the notifications configmap
kubectl apply -f argocd-notifications-configmap.yaml

# Apply the encrypted secret
sops -d argocd-notifications-secret.enc.yaml | kubectl apply -f -

# Restart ArgoCD components
kubectl rollout restart deployment/argocd-notifications-controller -n argocd
kubectl rollout restart deployment/argocd-repo-server -n argocd

# Check notification pod status
kubectl get pods -n argocd | grep notifications
```

## Testing

### 1. Test SOPS Encryption/Decryption

```bash
# Test encryption
echo "test: secret-value" | sops -e --input-type yaml --output-type yaml /dev/stdin

# Test decryption
sops -d argocd-notifications-secret.enc.yaml
```

### 2. Test Notifications

1. Create an out-of-sync condition in ArgoCD UI
2. Fix the application to trigger resolution notifications
3. Check Slack channel for notifications

### 3. Check Logs

```bash
# Check notifications controller logs
kubectl logs -n argocd deployment/argocd-notifications-controller

# Check repo-server logs for ksops activity
kubectl logs -n argocd deployment/argocd-repo-server | grep -i ksops
```

## Troubleshooting

### Common Issues

1. **SOPS "metadata not found" error**
   - File is not properly encrypted
   - Re-encrypt the file: `sops -e input.yaml > output.enc.yaml`

2. **Age key not found**
   - Check key location: `~/.config/sops/age/keys.txt`
   - Verify SOPS config matches your age public key

3. **Notifications not working**
   - Check secret is applied: `kubectl get secret argocd-notifications-secret -n argocd -o yaml`
   - Verify webhook URL and token are correct
   - Check notifications controller logs

4. **KSOPS plugin not working**
   - Verify plugin configmap is applied
   - Check repo-server has restarted after plugin installation
   - Ensure ArgoCD is configured to use the plugin

### Log Commands

```bash
# Check ArgoCD application status
kubectl get applications -n argocd

# Check notifications controller
kubectl logs -n argocd deployment/argocd-notifications-controller

# Check notifications controller with | tail -10
kubectl logs -n argocd deployment/argocd-notifications-controller
 | tail -10
# Check repo-server
kubectl logs -n argocd deployment/argocd-repo-server

# Check secret content
kubectl get secret argocd-notifications-secret -n argocd -o yaml
```

## Security Best Practices

1. **Never commit unencrypted secrets**
   - Always use `.enc.yaml` extension for encrypted files
   - Add unencrypted files to `.gitignore`

2. **Protect Age keys**
   - Store age keys securely
   - Backup keys in secure location
   - Use different keys for different environments

3. **Regular key rotation**
   - Rotate Slack tokens/webhooks periodically
   - Update encrypted secrets when keys change

4. **Monitor access**
   - Audit who has access to age keys
   - Monitor SOPS decryption activity
   - Review notification channel membership

## File Structure

```
argocd/
├── .sops.yaml                              # SOPS configuration
├── argocd-notifications-configmap.yaml     # Notifications configuration
├── argocd-notifications-secret.enc.yaml    # Encrypted Slack credentials
├── ksops-plugin-configmap.yaml            # KSOPS plugin configuration
├── argocd.yaml                             # Main ArgoCD configuration
└── README-ksops-slack-setup.md            # This documentation
```

## Conclusion

This setup provides:
- ✅ Secure secret management with SOPS/Age encryption
- ✅ Automated Slack notifications for application issues
- ✅ Resolution notifications when problems are fixed
- ✅ Proper GitOps workflow with encrypted secrets
- ✅ ArgoCD integration with KSOPS plugin

The implementation ensures that sensitive data is encrypted at rest and in transit, while providing comprehensive monitoring of application health and sync status through Slack notifications.