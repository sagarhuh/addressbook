# GitLab CI/CD Pipeline Setup for JAR Deployment

## Overview
This GitLab CI/CD pipeline automatically downloads 3 JAR files from your Nexus repository and deploys them to your application server whenever code is pushed to main/master/develop branches.

## Pipeline Stages
1. **Download**: Downloads JAR files from Nexus repository
2. **Deploy**: Copies JAR files to application server (manual trigger by default)
3. **Verify**: Verifies successful deployment

## Required GitLab Variables

You need to configure these variables in your GitLab project settings (Settings > CI/CD > Variables):

### Nexus Configuration
- `NEXUS_BASE_URL`: Your Nexus server URL (e.g., `https://nexus.yourcompany.com`)
- `NEXUS_REPO_NAME`: Repository name in Nexus (e.g., `maven-releases`)
- `NEXUS_USER`: Username for Nexus authentication
- `NEXUS_PASS`: Password for Nexus authentication (mark as protected/masked)

### JAR Configuration
Update the following variables in the `.gitlab-ci.yml` file with your actual JAR details:

```yaml
# JAR 1
JAR1_GROUP_ID: "com.yourcompany"           # e.g., "com.example.myapp"
JAR1_ARTIFACT_ID: "your-app-core"          # e.g., "core-service"
JAR1_VERSION: "${JAR1_VERSION:-1.0.0}"     # Can be overridden via variable

# JAR 2
JAR2_GROUP_ID: "com.yourcompany"
JAR2_ARTIFACT_ID: "your-app-service"
JAR2_VERSION: "${JAR2_VERSION:-1.0.0}"

# JAR 3
JAR3_GROUP_ID: "com.yourcompany"
JAR3_ARTIFACT_ID: "your-app-web"
JAR3_VERSION: "${JAR3_VERSION:-1.0.0}"
```

### Application Server Configuration
- `DEPLOY_HOST`: IP address or hostname of your application server
- `DEPLOY_USER`: SSH username for the application server
- `DEPLOY_PATH`: Target path on the server (default: `/opt/application/lib`)
- `DEPLOY_SSH_KEY`: Private SSH key for server access (mark as protected/masked)

### Optional Version Override Variables
- `JAR1_VERSION`: Override version for JAR 1
- `JAR2_VERSION`: Override version for JAR 2
- `JAR3_VERSION`: Override version for JAR 3

## SSH Key Setup

1. Generate SSH key pair on your local machine:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "gitlab-ci@yourproject"
   ```

2. Add the public key to your application server:
   ```bash
   ssh-copy-id -i ~/.ssh/id_rsa.pub user@your-server.com
   ```

3. Add the private key content to GitLab variable `DEPLOY_SSH_KEY`

## Application Server Requirements

Your application server should have:
- SSH access configured
- `sudo` privileges for the deployment user
- Target deployment directory created with appropriate permissions

## Pipeline Behavior

### Automatic Triggers
- Downloads JAR files automatically when code is pushed to main/master/develop branches
- Deployment stage is set to **manual** by default for safety

### Manual Deployment
- Go to GitLab > CI/CD > Pipelines
- Click on the pipeline run
- Click "Play" button on the `deploy_to_server` job

### Automatic Deployment
To enable automatic deployment, remove this line from the deploy job:
```yaml
when: manual  # Remove this line for automatic deployment
```

## Features

### Backup System
- Automatically backs up existing JAR files before deployment
- Backups are stored in `$APP_SERVER_PATH/backup/YYYYMMDD_HHMMSS/`

### Rollback Support
- Manual rollback job available
- Restores the most recent backup
- Triggered manually from GitLab UI

### Verification
- Verifies JAR file integrity after download
- Confirms successful deployment on the server
- Checks file sizes and existence

## Customization

### Service Management
If your application runs as a service, uncomment these lines in the deploy job:
```yaml
# Stop service before deployment
- ssh $APP_SERVER_USER@$APP_SERVER_HOST "sudo systemctl stop your-app-service"

# Start service after deployment  
- ssh $APP_SERVER_USER@$APP_SERVER_HOST "sudo systemctl start your-app-service"
```

### File Permissions
The pipeline sets JAR files to be owned by `app:app` with `644` permissions. Modify these in the deploy script if needed:
```yaml
sudo chown app:app $APP_SERVER_PATH/*.jar
sudo chmod 644 $APP_SERVER_PATH/*.jar
```

### Branch Configuration
Currently configured for `main`, `master`, and `develop` branches. Modify the `only` sections to change this:
```yaml
only:
  - main
  - master
  - develop
  - your-branch-name
```

## Troubleshooting

### Common Issues
1. **SSH Connection Failed**: Check SSH key configuration and server access
2. **Nexus Authentication**: Verify username/password and repository permissions
3. **File Not Found**: Check JAR coordinates (groupId, artifactId, version)
4. **Permission Denied**: Ensure deployment user has sudo privileges

### Debug Mode
Add this to any job's script section for debugging:
```yaml
- set -x  # Enable debug output
```

## Security Best Practices
- Mark sensitive variables as protected and masked
- Use SSH keys instead of passwords
- Limit deployment user privileges on the server
- Regular rotation of credentials