# GCP Command Line  🚀

## 🔥 General Commands

- **Authentication**:

  ```bash
  gcloud auth login                  # Authenticate a user via a browser.
  gcloud auth list                   # List authenticated accounts.
  gcloud auth activate-service-account --key-file=<KEY_FILE>
  ```

- **Configuration**:

  ```bash
  gcloud config list                 # View current configuration.
  gcloud config set project <PROJECT_ID>   # Set active project.
  gcloud config set compute/region <REGION>
  gcloud config set compute/zone <ZONE>
  ```

---

## 💻 Compute Engine (VMs)

- **Create VM Instance**:

  ```bash
  gcloud compute instances create <INSTANCE_NAME> \
    --zone=<ZONE> \
    --machine-type=<MACHINE_TYPE> \
    --image-family=<IMAGE_FAMILY> \
    --image-project=<IMAGE_PROJECT>
  ```

- **List Instances**:

  ```bash
  gcloud compute instances list
  ```

- **Delete Instance**:

  ```bash
  gcloud compute instances delete <INSTANCE_NAME> --zone=<ZONE>
  ```

- **SSH into Instance**:

  ```bash
  gcloud compute ssh <INSTANCE_NAME> --zone=<ZONE>
  ```

---

## 🌐 Networking

- **Create Firewall Rule**:

  ```bash
  gcloud compute firewall-rules create <RULE_NAME> \
    --allow tcp:<PORT> \
    --target-tags=<TAG>
  ```

- **List Firewall Rules**:

  ```bash
  gcloud compute firewall-rules list
  ```

- **Delete Firewall Rule**:

  ```bash
  gcloud compute firewall-rules delete <RULE_NAME>
  ```

---

## 🌩️ Load Balancing

### ✅ Steps to Set Up a Load Balancer

1. **Create VM Instances (Backends)**

```bash
 gcloud compute instances create www1 www2 \
   --zone=us-central1-c \
   --tags=network-lb-tag \
   --machine-type=e2-small \
   --image-family=debian-11 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     service apache2 restart
     echo "<h3>Web Server: $(hostname)</h3>" | tee /var/www/html/index.html'
```

2. **Create Firewall Rule (Allow Traffic to Instances)**

```bash
 gcloud compute firewall-rules create www-firewall-network-lb \
   --target-tags=network-lb-tag \
   --allow=tcp:80
```

3. **Reserve Static IP Address (External)**

```bash
 gcloud compute addresses create network-lb-ip-1 \
   --region=us-east4
```

4. **Create Health Check (Verify Backend Health)**

```bash
 gcloud compute http-health-checks create basic-check
```

5. **Create Target Pool (Group of Instances)**

```bash
 gcloud compute target-pools create www-pool \
   --region=us-east4 \
   --http-health-check=basic-check
```

6. **Add Instances to Target Pool**

```bash
 gcloud compute target-pools add-instances www-pool \
   --instances=www1,www2 \
   --zone=us-east4-b
```

7. **Create Forwarding Rule (Direct Traffic to Target Pool)**

```bash
 gcloud compute forwarding-rules create www-rule \
   --region=us-central1 \
   --ports=80 \
   --address=network-lb-ip-1 \
   --target-pool=www-pool
```

8. **Test the Load Balancer**

```bash
 gcloud compute forwarding-rules describe www-rule --region=us-east4 --format="get(IPAddress)"
```

Visit the IP address in your browser to confirm it works and is load balancing traffic between your instances.

---

## 🔑 IAM & Permissions

- **List Projects**:

  ```bash
  gcloud projects list
  ```

- **Add IAM Policy Binding**:

  ```bash
  gcloud projects add-iam-policy-binding <PROJECT_ID> \
    --member=<MEMBER> \
    --role=<ROLE>
  ```

- **View IAM Policies**:

  ```bash
  gcloud projects get-iam-policy <PROJECT_ID>
  ```

---

## 📦 Storage

- **Create a Storage Bucket**:

  ```bash
  gcloud storage buckets create gs://<BUCKET_NAME> --location=<REGION>
  ```

- **List Buckets**:

  ```bash
  gcloud storage buckets list
  ```

- **Upload File to Bucket**:

  ```bash
  gcloud storage cp <LOCAL_FILE_PATH> gs://<BUCKET_NAME>
  ```

- **Delete Bucket**:

  ```bash
  gcloud storage buckets delete gs://<BUCKET_NAME>
  ```

---

## 📋 Other Useful Commands

- **List All Resources in Project**:

  ```bash
  gcloud compute resources list
  ```

- **Set Default Project**:

  ```bash
  gcloud config set project <PROJECT_ID>
  ```

- **Delete Unused IP Addresses**:

  ```bash
  gcloud compute addresses delete <ADDRESS_NAME> --region=<REGION>
  ```

---

## 📚 Tips

- Use `gcloud <COMMAND> --help` for details about any command.
- Use `gcloud components update` to keep SDK up-to-date.
- Remember to use `--quiet` (`-q`) for non-interactive scripting.

---

This cheat sheet will help you with the most common GCP tasks. Want me to add more sections, like Kubernetes (GKE) or Cloud Functions? 😊

