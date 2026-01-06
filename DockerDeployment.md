# Credebl SSI Platform Deployment Using Docker Compose

## Prerequisites

### • Install Docker and Docker Compose
See: https://docs.docker.com/engine/install/

### • Install Node.js
Version: >= 18.17.0  
See: https://nodejs.dev/en/learn/how-to-install-nodejs/

### • Install NestJS CLI
```bash
npm i @nestjs/cli@latest 
```

## Setup Instructions
Navigate to the install/docker-deployment directory
Grant the setup script i.e., setup.sh execute permissions
Run the setup script
```bash
cd install/docker-deployment/
chmod +x setup.sh
./setup.sh
```

### Common challenges
You may encounter the following error message while this script executes on a remote server such as a Digital Ocean Droplet
```bash
│ Error: error initializing keycloak provider

│ 

│   with provider["registry.terraform.io/mrparkers/keycloak"],

│   on provider.tf line 1, in provider "keycloak":

│    1: provider "keycloak" {

│ 

│ failed to perform initial login to Keycloak: error sending POST request to

│ http://YOUR_DROPLET_PUBLIC_IP:8080/realms/master/protocol/openid-connect/token: 403

│ Forbidden

╵

❌ Terraform apply failed
```

The first thing you should do is review the logs.
```bash
docker logs $(docker ps -lq) --tail 50
```

#### The Problem

This error (403 Forbidden on the token endpoint) is a classic issue when deploying Keycloak 25+ (Quarkus) on a raw Public IP (like a DigitalOcean droplet) without SSL/HTTPS.
Even though Keycloak is running in "dev mode," it detects that the request is coming from an external IP address (YOUR_DROPLET_PUBLIC_IP) rather than localhost.
By default, Keycloak blocks non-HTTPS requests to the admin API from external sources for security reasons.

#### The Solution

While deploying on a raw IP (http://YOUR_DROPLET_PUBLIC_IP) and not a domain with an SSL certificate yet, you must explicitly tell Keycloak to allow "insecure" HTTP traffic on the public hostname.

You need to add several environment variables to the Keycloak service definition in your docker-compose.yaml file to disable these strict checks.

##### Edit the Docker Compose file
Ensure you are in the deployment directory:
```bash
cd ~/install/docker-deployment
```
Open setup.sh (or the specific setup/compose file used, sometimes named docker-compose.yaml, docker-compose-keycloak.yaml or just docker-compose.yml) using nano:
```bash
nano setup.sh
```
Locate the keycloak service block.
To search using nano
```bash
Ctrl+W
```
```bash
deploy_keycloak
```

Under the environment: section for Keycloak, add the following lines:
```bash
-e KC_HOSTNAME_STRICT=false \
-e KC_HOSTNAME_STRICT_HTTPS=false \
-e KC_HTTP_ENABLED=true \
-e KC_FEATURES=hostname:v1 \
```
Ensure the indentation matches the other environment variables exactly.
Ensure you include the backslash \ at the end of the new lines to keep the command chain intact.

| Variable | Value |	Why?  |
|---| --- | --- |
| KC_HOSTNAME_STRICT |	false |	Prevents Keycloak from validating the hostname (allows IP access) |
| KC_HOSTNAME_STRICT_HTTPS |	false |	Critical: Allows the admin API to accept login tokens over HTTP without SSL |
| KC_FEATURES | hostname:v1 | Switches Keycloak back to the configuration mode that understands "disable strict HTTPS." |

In some situations just modifying the script to indlude this may suffice, however, the 403 Forbidden error may persist because of a second security layer: The "Master" Realm Default Policy.
By default, the Master Realm in Keycloak is configured to require SSL for all external IP addresses. Even though you disabled the server-level HTTPS check, the realm-level policy sees you connecting from YOUR_DROPLET_PUBLIC_IP (which is an "external" IP) using HTTP, so it blocks the login.

We need to force Terraform to connect via localhost (127.0.0.1). Keycloak always trusts localhost and does not require SSL for it.
Edit setup.sh again
Open the script:
```bash
nano setup.sh
```
Press Ctrl+W and search for: setup_keycloak_terraform
Look for the line that defines NEW_URL. It typically looks like this:
```bash
NEW_URL="\"http://${MACHINE_IP}:${USED_PORT_KEYCLOAK}\""
```
Change it to use localhost instead:
```bash
NEW_URL="\"http://localhost:${USED_PORT_KEYCLOAK}\""
```

Re-run the setup
Save and exit (Ctrl+O, Enter, Ctrl+X), then run:
```bash
./setup.sh
```
