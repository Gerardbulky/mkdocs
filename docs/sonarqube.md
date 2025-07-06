# 🔧 SonarQube GitHub Integration Setup

A complete guide to setting up GitHub App integration with SonarQube for seamless code quality monitoring.

---

## 🚀 Overview

This guide will walk you through creating a GitHub App and configuring it with SonarQube to enable:
- 📊 Automated code quality checks on pull requests
- 🔄 Continuous integration with your GitHub repositories
- 📈 Quality gate status reporting

---

## 📋 Prerequisites

- ✅ Admin access to your GitHub organization
- ✅ SonarQube instance with administrator privileges
- ✅ Access to your SonarQube server URL

---

## 🎯 Step 1: Create GitHub App

### 1.1 Navigate to GitHub Developer Settings

🔗 **Visit:** [GitHub Apps Settings](https://github.com/settings/apps)

Click the **"New GitHub App"** button to begin.

### 1.2 Configure App Details

Fill out the GitHub App form with the following configuration:

| 📝 Field | 💡 Value |
|----------|----------|
| **GitHub App name** | `sonarqube-app` (or your preferred name) |
| **Homepage URL** | Your SonarQube URL<br>*Example: `https://sonarqube.yourcompany.com`* |
| **Callback URL** | `https://sonarqube.yourcompany.com/` |
| ✅ | Mark **Webhook Active** Only if you want to recieve webhook events |
| **Webhook URL** | `https://sonarqube.yourcompany.com/api/github/webhook` |
| **Webhook secret** | Leave blank *(optional)* |
| **Enable SSL Verification** if HTTPS else Don't Enable |
| **Generate Private Key** |

### 1.3 Set Repository Permissions
Go to **Permissions & Events**, 

In **Repository Permissions**, Configure the following permissions:

- **Checks:** Read and write ✅ 
- **Contents:** Read-only ✅
- **Issues:** Read-only *(if needed)* ✅
- **Metadata:** Read-only ✅
- **Pull requests:** Try Read-only if doesn't work, then write ✅

### 1.4 Subscribe to Events

Enable these webhook events:

- ✅ **Check run**
- ✅ **Check suite**
- ✅ **Pull request**
- ✅ **Push**

### 1.5 Installation Settings
- Go to **installed Apps** 
- Choose the GitHub org or repo
- Grant access to **All Repositories** or select repos

Click **"Save"** to save changes.

---

## 🔑 Step 2: Collect Required Credentials

After creating your GitHub App, gather these essential credentials:

| 🎯 SonarQube Field | 📍 GitHub Location |
|-------------------|-------------------|
| **GitHub App ID** | ✅ |
| **Client ID** | ✅ |
| **Client Secret** | ✅ |
| **Private Key (.pem)** | ✅ |

> 🔐 **Security Note:** Open the `.pem` file with a text editor and copy its entire content for use in SonarQube.

---

## ⚙️ Step 3: Configure SonarQube Integration

### 3.1 Access SonarQube Admin Panel

Navigate to: **Administration → DevOps Platforms → GitHub**

### 3.2 Enter Configuration Details

| 🎯 SonarQube Field | 📋 Example Value |
|-------------------|------------------|
| **Configuration name** | `my-gitHub-integration` |
| **GitHub API URL** | `https://api.github.com/` |
| **GitHub App ID** | `123456` *(from GitHub App page)* |
| **Client ID** | *✅* |
| **Client Secret** | *✅* |
| **Private Key** | *(paste full contents of .pem file)* |
| **Webhook Secret** | *(optional - only for code scanning alerts)* |

---

---

## 💬 Slack Integration Setup

Configure SonarQube to send quality gate alerts and notifications directly to your Slack channels with clickable links to the SonarQube UI.

### 🔧 Step 1: Create Slack App

#### 1.1 Create New Slack App

1. 🔗 **Visit:** [Slack API Apps](https://api.slack.com/apps)
2. ➕ **Click "Create New App"**
3. 📝 **Choose "From scratch"**
4. 📋 **Enter App Details:**
   - **App Name:** `SonarQube Alerts`
   - **Workspace:** Select your Slack workspace

#### 1.2 Configure Incoming Webhooks

1. 🔗 **Navigate to "Incoming Webhooks"** in your app settings
2. ✅ **Toggle "Activate Incoming Webhooks"** to ON
3. ➕ **Click "Add New Webhook to Workspace"**
4. 📍 **Select the channel** where you want alerts (e.g., `#code-quality`)
5. 📋 **Copy the webhook URL** (you'll need this for SonarQube)

> 📝 **Example webhook URL:** `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX`

### 🔧 Step 2: Install SonarQube Slack Plugin

#### 2.1 Download and Install Plugin

```bash
# Download the Slack plugin
wget https://github.com/sonarqube-plugins/sonar-slack-notifier-plugin/releases/latest/download/sonar-slack-notifier-plugin.jar

# Move to SonarQube plugins directory
sudo mv sonar-slack-notifier-plugin.jar /opt/sonarqube/extensions/plugins/

# Restart SonarQube
sudo systemctl restart sonarqube
```

#### 2.2 Alternative: Manual Plugin Installation

1. 🔗 **Download from:** [SonarQube Slack Plugin](https://github.com/sonarqube-plugins/sonar-slack-notifier-plugin)
2. 📁 **Copy JAR file** to `/opt/sonarqube/extensions/plugins/`
3. 🔄 **Restart SonarQube service**

### 🔧 Step 3: Configure Slack in SonarQube

#### 3.1 Access SonarQube Administration

1. 🔗 **Login to SonarQube:** `https://sonarqube.yourdomain.com`
2. 🔧 **Navigate to:** Administration → Configuration → Slack

#### 3.2 Global Slack Configuration

| 🎯 Setting | 📋 Value |
|-----------|----------|
| **Webhook URL** | `https://hooks.slack.com/services/YOUR/WEBHOOK/URL` |
| **Channel** | `#code-quality` (or your preferred channel) |
| **Username** | `SonarQube` |
| **Icon** | `:warning:` or custom emoji |

### 🔧 Step 4: Configure Project-Level Notifications

#### 4.1 Per-Project Slack Settings

For each project that should send Slack notifications:

1. 🔗 **Navigate to:** Project → Project Settings → Slack
2. 📝 **Configure the following:**

| 🎯 Setting | 📋 Value | 📝 Description |
|-----------|----------|----------------|
| **Enabled** | ✅ | Enable Slack notifications |
| **Channel** | `#project-alerts` | Project-specific channel |
| **Quality Gate** | ✅ | Send quality gate status |
| **New Issues** | ✅ | Alert on new issues |
| **Include Branch** | ✅ | Show branch information |
| **SonarQube URL** | `https://sonarqube.yourdomain.com` | Base URL for links |

### 🔧 Step 5: Advanced Webhook Configuration

#### 5.1 Custom Webhook Script

Create a custom webhook script for enhanced notifications:

```bash
# Create custom webhook script
sudo nano /opt/sonarqube/bin/slack-webhook.sh
```

Add this enhanced webhook script:

```bash
#!/bin/bash

# Enhanced SonarQube Slack Webhook
# Usage: slack-webhook.sh <webhook_url> <channel> <message> <project_key> <project_name>

WEBHOOK_URL="$1"
CHANNEL="$2"
MESSAGE="$3"
PROJECT_KEY="$4"
PROJECT_NAME="$5"
SONARQUBE_URL="https://sonarqube.yourdomain.com"

# Quality Gate Status Colors
case "$MESSAGE" in
    *"PASSED"*) COLOR="good" ;;
    *"FAILED"*) COLOR="danger" ;;
    *"WARNING"*) COLOR="warning" ;;
    *) COLOR="good" ;;
esac

# Extract quality gate status
if [[ "$MESSAGE" =~ "Quality Gate: "([A-Z]+) ]]; then
    QG_STATUS="${BASH_REMATCH[1]}"
else
    QG_STATUS="UNKNOWN"
fi

# Create rich Slack message
SLACK_PAYLOAD=$(cat <<EOF
{
    "channel": "$CHANNEL",
    "username": "SonarQube",
    "icon_emoji": ":sonarqube:",
    "attachments": [
        {
            "color": "$COLOR",
            "title": "🔍 SonarQube Quality Gate: $QG_STATUS",
            "title_link": "$SONARQUBE_URL/dashboard?id=$PROJECT_KEY",
            "text": "$MESSAGE",
            "fields": [
                {
                    "title": "Project",
                    "value": "$PROJECT_NAME",
                    "short": true
                },
                {
                    "title": "Status",
                    "value": "$QG_STATUS",
                    "short": true
                }
            ],
            "actions": [
                {
                    "type": "button",
                    "text": "📊 View Dashboard",
                    "url": "$SONARQUBE_URL/dashboard?id=$PROJECT_KEY"
                },
                {
                    "type": "button",
                    "text": "🐛 View Issues",
                    "url": "$SONARQUBE_URL/project/issues?id=$PROJECT_KEY"
                },
                {
                    "type": "button",
                    "text": "📈 View Measures",
                    "url": "$SONARQUBE_URL/component_measures?id=$PROJECT_KEY"
                }
            ],
            "footer": "SonarQube Quality Gate",
            "ts": $(date +%s)
        }
    ]
}
EOF
)

# Send to Slack
curl -X POST -H 'Content-type: application/json' \
    --data "$SLACK_PAYLOAD" \
    "$WEBHOOK_URL"
```

Make the script executable:

```bash
sudo chmod +x /opt/sonarqube/bin/slack-webhook.sh
```

### 🔧 Step 6: Configure Quality Gates with Slack

#### 6.1 Quality Gate Webhook Configuration

1. 🔗 **Navigate to:** Quality Gates → Your Quality Gate → Webhooks
2. ➕ **Add Webhook** with these details:

| 🎯 Field | 📋 Value |
|----------|----------|
| **Name** | `Slack Notifications` |
| **URL** | `https://hooks.slack.com/services/YOUR/WEBHOOK/URL` |
| **Secret** | *(optional)* |

#### 6.2 Webhook Payload Template

Configure a custom payload template for rich Slack messages:

```json
{
    "channel": "#code-quality",
    "username": "SonarQube",
    "icon_emoji": ":sonarqube:",
    "attachments": [
        {
            "color": "{{#eq status.qualityGateStatus 'PASSED'}}good{{/eq}}{{#eq status.qualityGateStatus 'FAILED'}}danger{{/eq}}{{#eq status.qualityGateStatus 'WARNING'}}warning{{/eq}}",
            "title": "🔍 Quality Gate: {{status.qualityGateStatus}}",
            "title_link": "{{serverUrl}}/dashboard?id={{project.key}}",
            "text": "Project: *{{project.name}}*\nBranch: {{branch.name}}\nStatus: *{{status.qualityGateStatus}}*",
            "fields": [
                {
                    "title": "📊 Coverage",
                    "value": "{{measures.coverage}}%",
                    "short": true
                },
                {
                    "title": "🐛 Issues",
                    "value": "{{measures.violations}}",
                    "short": true
                },
                {
                    "title": "🔒 Security",
                    "value": "{{measures.security_rating}}",
                    "short": true
                },
                {
                    "title": "🧹 Maintainability",
                    "value": "{{measures.maintainability_rating}}",
                    "short": true
                }
            ],
            "actions": [
                {
                    "type": "button",
                    "text": "📊 View Dashboard",
                    "url": "{{serverUrl}}/dashboard?id={{project.key}}"
                },
                {
                    "type": "button",
                    "text": "🐛 View Issues",
                    "url": "{{serverUrl}}/project/issues?id={{project.key}}"
                }
            ],
            "footer": "SonarQube Analysis",
            "ts": {{analysisDate}}
        }
    ]
}
```

### 🔧 Step 7: Configure Branch-Specific Notifications

#### 7.1 Branch Pattern Configuration

Set up notifications for specific branches:

```bash
# Configure branch-specific Slack channels
# Main branch -> #releases
# Feature branches -> #dev-alerts
# Pull requests -> #pr-reviews
```

#### 7.2 SonarQube Branch Configuration

1. 🔗 **Navigate to:** Administration → Configuration → Branches and Pull Requests
2. 📝 **Configure branch patterns:**

| 🎯 Branch Pattern | 📍 Slack Channel | 🔔 Notification Type |
|------------------|------------------|---------------------|
| `main` or `master` | `#releases` | All notifications |
| `develop` | `#dev-alerts` | Quality gate failures |
| `feature/*` | `#dev-alerts` | Major issues only |
| `PR-*` | `#pr-reviews` | Quality gate status |

### 🔧 Step 8: Test Slack Integration

#### 8.1 Manual Test

```bash
# Test webhook directly
curl -X POST -H 'Content-type: application/json' \
    --data '{
        "channel": "#code-quality",
        "username": "SonarQube",
        "text": "🧪 Testing SonarQube Slack integration!",
        "attachments": [
            {
                "color": "good",
                "title": "Test Notification",
                "title_link": "https://sonarqube.yourdomain.com",
                "text": "This is a test message from SonarQube"
            }
        ]
    }' \
    https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

#### 8.2 Trigger Analysis

Run a SonarQube analysis to test the integration:

```bash
# Run analysis with Slack notification
mvn sonar:sonar \
    -Dsonar.projectKey=your-project-key \
    -Dsonar.host.url=https://sonarqube.yourdomain.com \
    -Dsonar.login=your-token
```

### 📊 Step 9: Custom Notification Templates

#### 9.1 Quality Gate Passed Template

```json
{
    "channel": "#code-quality",
    "username": "SonarQube",
    "icon_emoji": ":white_check_mark:",
    "attachments": [
        {
            "color": "good",
            "title": "✅ Quality Gate PASSED",
            "title_link": "{{serverUrl}}/dashboard?id={{project.key}}",
            "text": "Great job! Your code quality meets our standards.",
            "fields": [
                {
                    "title": "Project",
                    "value": "<{{serverUrl}}/dashboard?id={{project.key}}|{{project.name}}>",
                    "short": true
                },
                {
                    "title": "Branch",
                    "value": "{{branch.name}}",
                    "short": true
                }
            ],
            "actions": [
                {
                    "type": "button",
                    "text": "📊 View Dashboard",
                    "url": "{{serverUrl}}/dashboard?id={{project.key}}"
                }
            ]
        }
    ]
}
```

#### 9.2 Quality Gate Failed Template

```json
{
    "channel": "#code-quality",
    "username": "SonarQube",
    "icon_emoji": ":x:",
    "attachments": [
        {
            "color": "danger",
            "title": "❌ Quality Gate FAILED",
            "title_link": "{{serverUrl}}/dashboard?id={{project.key}}",
            "text": "⚠️ Code quality issues detected. Please review and fix.",
            "fields": [
                {
                    "title": "Project",
                    "value": "<{{serverUrl}}/dashboard?id={{project.key}}|{{project.name}}>",
                    "short": true
                },
                {
                    "title": "Branch",
                    "value": "{{branch.name}}",
                    "short": true
                },
                {
                    "title": "Failed Conditions",
                    "value": "{{#each qualityGate.conditions}}{{#if status.failed}}• {{metric.name}}: {{status.actualValue}} (threshold: {{status.expectedValue}})\n{{/if}}{{/each}}",
                    "short": false
                }
            ],
            "actions": [
                {
                    "type": "button",
                    "text": "🔍 View Issues",
                    "url": "{{serverUrl}}/project/issues?id={{project.key}}&resolved=false"
                },
                {
                    "type": "button",
                    "text": "📊 View Dashboard",
                    "url": "{{serverUrl}}/dashboard?id={{project.key}}"
                }
            ]
        }
    ]
}
```

### ✅ Verification Steps

1. 🔄 **Test webhook** by triggering a SonarQube analysis
2. 📱 **Check Slack channel** for notifications
3. 🔗 **Click on links** to verify they redirect to correct SonarQube pages
4. 📊 **Verify dashboard links** work correctly
5. 🐛 **Test issue links** navigate to the right project issues

---

## ✅ Next Steps

After completing the setup:

1. 🔄 **Test the connection** in SonarQube
2. 🏗️ **Configure project bindings** for your repositories
3. 📊 **Set up quality gates** for your projects
4. 💬 **Configure Slack notifications** for your teams
5. 🎉 **Create a pull request** to see the integration in action!

---

## 🆘 Troubleshooting

- **Connection issues?** Verify your webhook URL is accessible
- **Permission errors?** Check that the GitHub App has the required repository permissions
- **Authentication problems?** Ensure the private key is copied correctly with proper formatting

---

*🤖 Happy coding with automated quality checks!*