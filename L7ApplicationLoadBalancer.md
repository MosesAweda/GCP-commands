# Setting Up a Load-Balanced Web Server on Google Cloud Platform

This guide walks through creating a load-balanced web server setup on Google Cloud Platform using the `gcloud` command-line tools.

## Step 1: Create an Instance Template

```bash
gcloud compute instance-templates create lb-backend-template \
   --region=us-central1 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```

**Explanation**: This command creates a template for VM instances that will serve as your backend servers. The template:
- Uses the `us-central1` region
- Connects to the default network and subnet
- Applies the tag `allow-health-check` (used later for firewall rules)
- Uses an `e2-medium` machine type (2 vCPUs, 4GB memory)
- Runs Debian 11 as the operating system
- Includes a startup script that:
  - Updates the package list
  - Installs Apache web server
  - Enables SSL site and module
  - Creates a simple web page showing the VM's hostname
  - Restarts Apache to apply changes

## Step 2: Create a Managed Instance Group

```bash
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=us-central1-c
```

**Explanation**: This creates a managed instance group named `lb-backend-group` that:
- Uses the template created in Step 1
- Creates 2 VM instances
- Places them in the `us-central1-c` zone
- Automatically manages these instances (handles recreation if they fail)

## Step 3: Create a Firewall Rule for Health Checks

```bash
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

**Explanation**: This creates a firewall rule that:
- Allows incoming traffic to your instances
- Only allows traffic from Google's health checking systems (IP ranges 130.211.0.0/22 and 35.191.0.0/16)
- Targets instances with the tag `allow-health-check` (set in Step 1)
- Allows TCP traffic on port 80 (HTTP)

## Step 4: Reserve a Static External IP Address

```bash
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```

**Explanation**: This reserves a static external IPv4 address that will be used for your load balancer, making it accessible from the internet with a consistent IP address.

## Step 5: View the Reserved IP Address (Optional)

```bash
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```

**Explanation**: This displays the actual IP address that was reserved in Step 4. You might want to note this for future reference.

## Step 6: Create a Health Check

```bash
gcloud compute health-checks create http http-basic-check \
  --port 80
```

**Explanation**: This creates an HTTP health check named `http-basic-check` that:
- Periodically checks your instances on port 80
- Determines if they are healthy and can receive traffic
- Automatically removes unhealthy instances from the load balancer

## Step 7: Create a Backend Service

```bash
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```

**Explanation**: This creates a backend service named `web-backend-service` that:
- Uses HTTP protocol
- Listens on named port "http"
- Uses the health check created in Step 6
- Is globally available (not restricted to a specific region)

## Step 8: Add Your Instance Group to the Backend Service

```bash
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=us-central1-c \
  --global
```

**Explanation**: This connects your managed instance group to the backend service:
- References the instance group created in Step 2
- Specifies the zone where the instance group is located
- Updates the global backend service

## Step 9: Create a URL Map

```bash
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
```

**Explanation**: This creates a URL map named `web-map-http` that:
- Routes incoming requests to appropriate backend services
- Uses `web-backend-service` as the default service for all traffic
- Could be configured to route different URLs to different backends (not used in this basic setup)

## Step 10: Create an HTTP Proxy

```bash
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```

**Explanation**: This creates an HTTP proxy named `http-lb-proxy` that:
- References the URL map created in Step 9
- Acts as the frontend of your load balancer
- Processes incoming HTTP requests

## Step 11: Create a Forwarding Rule

```bash
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1 \
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```

**Explanation**: This creates a forwarding rule named `http-content-rule` that:
- Uses the static IP address reserved in Step 4
- Is globally available
- Routes traffic to the HTTP proxy created in Step 10
- Listens on port 80 (HTTP)

## Complete Setup

At this point, your load-balanced web server infrastructure is complete. The system will:
1. Accept HTTP requests at the IP address you reserved
2. Route those requests through the load balancer
3. Distribute traffic across your two VM instances
4. Automatically perform health checks on your instances
5. Remove unhealthy instances from the rotation if they fail checks
6. Each instance will serve a simple page showing which VM is handling the request