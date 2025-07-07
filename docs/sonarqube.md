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
# Check if Python is installed
python3 --version
pip --version

# If not install using:
sudo apt update
sudo apt install python3 python3-pip -y

# Set Python to point to Python3
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3 1
```
```bash
# Install Python dependencies
pip install flask requests

# Or create a requirements.txt file
cat > requirements.txt << EOF
flask==2.3.3
requests==2.31.0
EOF

# 1. Create a virtual environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Create the Webhook Handler:**
```bash
# Create the webhook handler script
sudo nano /opt/sonarqube-slack-handler/webhook_handler.py
```

**Python Webhook Handler Script:**

The current setup with the Python Flask webhook handler is actually more robust and flexible than the plugin-based approach.
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
    if "YOUR/WEBHOOK/URL from GitHub developer settings" in SLACK_WEBHOOK_URL:
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
User=ubuntu   #Change user to your server user
#Group=sonarqube
WorkingDirectory=/opt/sonarqube-slack-handler
ExecStart=/home/ubuntu/venv/bin/python3 /opt/sonarqube-slack-handler/webhook_handler.py #running 'which python3' will show the python path. /home/ubuntu/venv/bin/python3 
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Start the Service:**
```bash
# Create directory and set permissions
sudo chown ubuntu:ubuntu /opt/sonarqube-slack-handler # Ubuntu user can run the script in the directory

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable sonarqube-slack-handler
sudo systemctl start sonarqube-slack-handler

# Check status
sudo systemctl status sonarqube-slack-handler
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




### 🔧 Step 7: Configure Branch-Specific Notifications

⚠️ **Important Note:** Branch and Pull Request analysis is only available in **SonarQube Developer Edition and higher**. If you're using Community Edition, you can still implement branch-specific routing in your Python webhook handler.

#### 7.1 SonarQube Edition Differences

| Feature | Community Edition | Developer Edition+ |
|---------|------------------|-------------------|
| **Branch Analysis** | ❌ Main branch only | ✅ All branches |
| **Pull Request Analysis** | ❌ Not available | ✅ Full PR analysis |
| **Branch Configuration UI** | ❌ Not available | ✅ Available under Administration |
| **Webhook Branch Data** | ❌ Limited | ✅ Full branch information |

#### 7.2 Community Edition: Python Handler Branch Routing

Since the webhook data in Community Edition is limited, implement branch detection in your Python handler:

```python
def get_branch_from_project_key(project_key):
    """Extract branch info from project key or other sources"""
    
    # Option 1: If you include branch in project key
    if '-feature-' in project_key:
        return 'feature'
    elif '-hotfix-' in project_key:
        return 'hotfix'
    elif project_key.endswith('-dev'):
        return 'develop'
    else:
        return 'main'

def get_slack_channel_community_edition(project_key, project_name):
    """Route notifications based on project naming or other indicators"""
    
    branch_type = get_branch_from_project_key(project_key)
    
    # Channel routing for Community Edition
    if branch_type == 'main':
        return '#releases'
    elif branch_type in ['feature', 'develop']:
        return '#dev-alerts'
    elif branch_type == 'hotfix':
        return '#urgent-fixes'
    else:
        return '#code-quality'

# Update your webhook handler:
def format_slack_message(webhook_data):
    project = webhook_data.get('project', {})
    project_key = project.get('key', 'unknown')
    project_name = project.get('name', 'Unknown Project')
    
    # Use Community Edition branch detection
    channel = get_slack_channel_community_edition(project_key, project_name)
    
    message = {
        "channel": channel,
        # ... rest of your message
    }
    return message
```

#### 7.3 Developer Edition+: Native Branch Configuration

If you upgrade to Developer Edition or higher, you'll have access to:

1. 🔗 **Navigate to:** Administration → Configuration → Branches and Pull Requests
2. 📝 **Configure branch patterns:**

| 🎯 Branch Pattern | 📍 Slack Channel | 🔔 Notification Type |
|------------------|------------------|---------------------|
| `main` or `master` | `#releases` | All notifications |
| `develop` | `#dev-alerts` | Quality gate failures |
| `feature/*` | `#dev-alerts` | Major issues only |
| `PR-*` | `#pr-reviews` | Quality gate status |

#### 7.4 Practical Implementation for Community Edition

Since you're using Community Edition, here are your best options:

**Option 1: Project-Based Routing**
Create separate SonarQube projects for different branches:
- `my-app-main` → Routes to `#releases`
- `my-app-develop` → Routes to `#dev-alerts`
- `my-app-feature-xyz` → Routes to `#dev-alerts`

**Option 2: Single Channel (Recommended)**
Keep it simple and use one channel (`#code-quality`) for all notifications. This works perfectly for most teams.

**Option 3: Enhanced Python Handler**
Add the branch detection code above to your existing `webhook_handler.py` file.

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