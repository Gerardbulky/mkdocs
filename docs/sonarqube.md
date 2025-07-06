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

### 🔧 Step 2: Choose Integration Method

⚠️ **Important Note:** There is no official SonarQube Slack plugin from SonarSource. We recommend using the built-in webhooks feature for the most reliable integration.

#### 2.1 Method 1: Built-in Webhooks (Recommended)

This is the official and most reliable method for SonarQube → Slack integration using SonarQube's native webhook feature with a custom handler:

✅ **Advantages:**
- Officially supported by SonarSource
- Works with all modern SonarQube versions
- No plugin maintenance required
- More reliable and secure
- Full control over message formatting
- Can add custom logic and filtering

**Choose one of these webhook approaches:**

**Option A: External Slack Handler Service**
- Deploy a lightweight web service (Node.js, Python Flask, etc.)
- Receives SonarQube webhooks and transforms them for Slack
- Can run on the same server or separate service
- Best for teams with multiple projects

**Option B: Simple Python Script**
- Lightweight Python script that processes webhooks
- Can run as a systemd service or via cron
- Perfect for single-server setups
- Minimal resource usage

**Continue to Step 3** for webhook implementation details.


### 🔧 Step 3: Webhook Implementation Options

#### 3.1 Option A: Python Flask Webhook Handler (Recommended)

Create a lightweight Python service to handle SonarQube webhooks:

**Install Dependencies:**
```bash
# Install Python dependencies
pip install flask requests

# Or create a requirements.txt file
cat > requirements.txt << EOF
flask==2.3.3
requests==2.31.0
EOF

pip install -r requirements.txt
```

**Create the Webhook Handler:**
```bash
# Create the webhook handler script
sudo nano /opt/sonarqube-slack-handler/webhook_handler.py
```

**Python Webhook Handler Script:**
```python
#!/usr/bin/env python3
"""
SonarQube to Slack Webhook Handler
Receives SonarQube webhooks and forwards formatted messages to Slack
"""

import json
import requests
from flask import Flask, request, jsonify
import logging
from datetime import datetime

app = Flask(__name__)

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Configuration
SLACK_WEBHOOK_URL = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
SONARQUBE_BASE_URL = "https://sonarqube.yourdomain.com"

def get_color_for_status(status):
    """Return Slack color based on quality gate status"""
    colors = {
        'OK': 'good',
        'PASSED': 'good',
        'ERROR': 'danger',
        'FAILED': 'danger',
        'WARN': 'warning',
        'WARNING': 'warning'
    }
    return colors.get(status.upper(), 'good')

def format_slack_message(webhook_data):
    """Transform SonarQube webhook data into Slack message format"""
    
    # Extract data from SonarQube webhook
    project = webhook_data.get('project', {})
    quality_gate = webhook_data.get('qualityGate', {})
    analysis_date = webhook_data.get('analysedAt', '')
    branch = webhook_data.get('branch', {})
    
    project_key = project.get('key', 'unknown')
    project_name = project.get('name', 'Unknown Project')
    qg_status = quality_gate.get('status', 'UNKNOWN')
    branch_name = branch.get('name', 'main')
    
    # Format analysis date
    if analysis_date:
        try:
            dt = datetime.fromisoformat(analysis_date.replace('Z', '+00:00'))
            formatted_date = dt.strftime('%Y-%m-%d %H:%M:%S UTC')
        except:
            formatted_date = analysis_date
    else:
        formatted_date = 'N/A'
    
    # Build dashboard URL
    dashboard_url = f"{SONARQUBE_BASE_URL}/dashboard?id={project_key}"
    issues_url = f"{SONARQUBE_BASE_URL}/project/issues?id={project_key}&resolved=false"
    measures_url = f"{SONARQUBE_BASE_URL}/component_measures?id={project_key}"
    
    # Create Slack message
    color = get_color_for_status(qg_status)
    
    # Status emoji
    status_emoji = "✅" if qg_status in ['OK', 'PASSED'] else "❌"
    
    message = {
        "channel": "#code-quality",
        "username": "SonarQube",
        "icon_emoji": ":sonarqube:",
        "attachments": [
            {
                "color": color,
                "title": f"{status_emoji} Quality Gate: {qg_status}",
                "title_link": dashboard_url,
                "text": f"Analysis completed for project *{project_name}*",
                "fields": [
                    {
                        "title": "Project",
                        "value": f"<{dashboard_url}|{project_name}>",
                        "short": True
                    },
                    {
                        "title": "Branch",
                        "value": branch_name,
                        "short": True
                    },
                    {
                        "title": "Status",
                        "value": qg_status,
                        "short": True
                    },
                    {
                        "title": "Analyzed",
                        "value": formatted_date,
                        "short": True
                    }
                ],
                "actions": [
                    {
                        "type": "button",
                        "text": "📊 View Dashboard",
                        "url": dashboard_url
                    },
                    {
                        "type": "button",
                        "text": "🐛 View Issues",
                        "url": issues_url
                    },
                    {
                        "type": "button",
                        "text": "📈 View Measures",
                        "url": measures_url
                    }
                ],
                "footer": "SonarQube Quality Gate",
                "ts": int(datetime.now().timestamp())
            }
        ]
    }
    
    return message

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    """Handle incoming SonarQube webhook"""
    try:
        # Get webhook data
        webhook_data = request.get_json()
        
        if not webhook_data:
            logger.error("No JSON data received")
            return jsonify({"error": "No data received"}), 400
        
        logger.info(f"Received webhook: {json.dumps(webhook_data, indent=2)}")
        
        # Format message for Slack
        slack_message = format_slack_message(webhook_data)
        
        # Send to Slack
        response = requests.post(
            SLACK_WEBHOOK_URL,
            json=slack_message,
            timeout=30
        )
        
        if response.status_code == 200:
            logger.info("Successfully sent message to Slack")
            return jsonify({"status": "success"}), 200
        else:
            logger.error(f"Failed to send to Slack: {response.status_code} - {response.text}")
            return jsonify({"error": "Failed to send to Slack"}), 500
            
    except Exception as e:
        logger.error(f"Error processing webhook: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    return jsonify({"status": "healthy", "timestamp": datetime.now().isoformat()})

if __name__ == '__main__':
    # Update the webhook URL before running
    if "YOUR/WEBHOOK/URL" in SLACK_WEBHOOK_URL:
        print("⚠️  Please update SLACK_WEBHOOK_URL in the script before running!")
        exit(1)
    
    # Run the Flask app
    app.run(host='0.0.0.0', port=5000, debug=False)
```

**Create Systemd Service:**
```bash
# Create systemd service file
sudo nano /etc/systemd/system/sonarqube-slack-handler.service
```

```ini
[Unit]
Description=SonarQube Slack Webhook Handler
After=network.target

[Service]
Type=simple
User=sonarqube
Group=sonarqube
WorkingDirectory=/opt/sonarqube-slack-handler
ExecStart=/usr/bin/python3 /opt/sonarqube-slack-handler/webhook_handler.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Start the Service:**
```bash
# Create directory and set permissions
sudo mkdir -p /opt/sonarqube-slack-handler
sudo chown sonarqube:sonarqube /opt/sonarqube-slack-handler

# Copy the script (update the SLACK_WEBHOOK_URL first!)
sudo cp webhook_handler.py /opt/sonarqube-slack-handler/

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable sonarqube-slack-handler
sudo systemctl start sonarqube-slack-handler

# Check status
sudo systemctl status sonarqube-slack-handler
```

#### 3.2 Option B: Simple Python Script Handler

For a simpler approach, use this lightweight script:

```python
#!/usr/bin/env python3
"""
Simple SonarQube to Slack Webhook Handler
Minimal script for basic Slack notifications
"""

import json
import requests
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler
import logging

# Configuration
SLACK_WEBHOOK_URL = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
SONARQUBE_URL = "https://sonarqube.yourdomain.com"
PORT = 5000

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path == '/webhook':
            try:
                # Read webhook data
                content_length = int(self.headers['Content-Length'])
                post_data = self.rfile.read(content_length)
                webhook_data = json.loads(post_data.decode('utf-8'))
                
                # Extract basic info
                project = webhook_data.get('project', {})
                quality_gate = webhook_data.get('qualityGate', {})
                
                project_name = project.get('name', 'Unknown')
                project_key = project.get('key', 'unknown')
                status = quality_gate.get('status', 'UNKNOWN')
                
                # Create simple Slack message
                color = 'good' if status in ['OK', 'PASSED'] else 'danger'
                emoji = '✅' if status in ['OK', 'PASSED'] else '❌'
                
                slack_message = {
                    "text": f"{emoji} SonarQube Quality Gate: *{status}*",
                    "attachments": [
                        {
                            "color": color,
                            "title": f"Project: {project_name}",
                            "title_link": f"{SONARQUBE_URL}/dashboard?id={project_key}",
                            "text": f"Quality gate status: *{status}*",
                            "actions": [
                                {
                                    "type": "button",
                                    "text": "View Dashboard",
                                    "url": f"{SONARQUBE_URL}/dashboard?id={project_key}"
                                }
                            ]
                        }
                    ]
                }
                
                # Send to Slack
                response = requests.post(SLACK_WEBHOOK_URL, json=slack_message)
                
                if response.status_code == 200:
                    self.send_response(200)
                    self.end_headers()
                    self.wfile.write(b'OK')
                    logger.info(f"Sent notification for {project_name}: {status}")
                else:
                    self.send_response(500)
                    self.end_headers()
                    logger.error(f"Failed to send to Slack: {response.status_code}")
                    
            except Exception as e:
                logger.error(f"Error: {e}")
                self.send_response(500)
                self.end_headers()
        else:
            self.send_response(404)
            self.end_headers()

if __name__ == '__main__':
    if "YOUR/WEBHOOK/URL" in SLACK_WEBHOOK_URL:
        print("Please update SLACK_WEBHOOK_URL before running!")
        sys.exit(1)
    
    server = HTTPServer(('0.0.0.0', PORT), WebhookHandler)
    logger.info(f"Starting webhook handler on port {PORT}")
    server.serve_forever()
```

#### 3.3 Configure SonarQube Webhooks

Now configure SonarQube to send webhooks to your handler:

1. 🔗 **Login to SonarQube:** `https://sonarqube.yourdomain.com`
2. 🔧 **Navigate to:** Administration → Configuration → Webhooks
3. ➕ **Add Webhook:**

| 🎯 Field | 📋 Value |
|----------|----------|
| **Name** | `Slack Notifications` |
| **URL** | `http://localhost:5000/webhook` (or your server IP) |
| **Secret** | *(optional)* |

#### 3.4 Test the Integration

```bash
# Test the webhook handler
curl -X POST http://localhost:5000/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "project": {
      "key": "test-project",
      "name": "Test Project"
    },
    "qualityGate": {
      "status": "PASSED"
    }
  }'

# Check handler logs
sudo journalctl -u sonarqube-slack-handler -f
```

### 🔧 Step 4: Configure Slack in SonarQube (Plugin Method)

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