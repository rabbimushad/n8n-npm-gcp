# How to Install and Run n8n Using npm on a Google Cloud Platform VM (Recommended for Dev/Test)

This guide will walk you through the process of setting up an n8n instance (an automation tool) using npm on a Google Cloud Platform (GCP) virtual machine. We will cover everything from creating the VM to configuring environment variables and running n8n with PM2 so that it keeps running even if your SSH session ends.

> **Note:** This guide assumes you are using a Linux-based VM (e.g., Ubuntu 20.04 or 22.04) on GCP.

---

## Step 1: Create a GCP VM Instance

1. **Log in to GCP Console:**
   - Go to [Google Cloud Console](https://console.cloud.google.com/) and sign in.

2. **Create a New VM Instance:**
   - Navigate to **Compute Engine > VM Instances**.
   - Click **Create Instance**.
   - Choose a Linux-based image (Ubuntu 20.04 LTS or Ubuntu 22.04 LTS are recommended).
   - Select an appropriate machine type (e.g., e2-micro for testing).
   - In the **Firewall** section, enable **Allow HTTP traffic** (and **HTTPS** if you plan to use TLS later).
   - Click **Create** and note the external IP address of your new VM.

---

## Step 2: Connect to Your VM and Update Packages

1. **SSH into the VM:**
   - Use the SSH button in the GCP Console or run the following command in your local terminal (replace `[YOUR_INSTANCE_NAME]` with your VM name):
     ```bash
     gcloud compute ssh [YOUR_INSTANCE_NAME]
     ```

2. **Update Your System:**
   - Once connected, run:
     ```bash
     sudo apt update && sudo apt upgrade -y
     ```

---

## Step 3: Install Node.js and npm

1. **Install Node.js (v18 is recommended):**
   ```bash
   curl -fsSL https://deb.nodesource.com/setup_current.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```

2. **Verify the Installation:**
   ```bash
   node -v
   npm -v
   ```
   You should see the versions of Node.js and npm printed on the screen.

---

## Step 4: Install n8n Globally via npm

1. **Install n8n:**
   ```bash
   sudo npm install -g n8n
   ```
   This command installs the n8n command-line tool globally so you can start it from anywhere.

2. **(Optional) Create a Data Directory:**
   - n8n stores its configuration and workflows in the `~/.n8n` folder by default. Create the directory (if it doesn’t already exist):
     ```bash
     mkdir -p ~/.n8n
     ```

---

## Step 5: Configure Environment Variables

You can configure n8n using environment variables. There are two main methods:

### Option A: Export in Your Shell

1. **Edit Your Shell Profile:**
   - Open your shell’s profile file (e.g., `~/.bashrc`):
     ```bash
     nano ~/.bashrc
     ```

2. **Add the Following Lines:**
   ```bash
   # Enable basic authentication for n8n
   export N8N_BASIC_AUTH_ACTIVE=true
   export N8N_BASIC_AUTH_USER=your_username
   export N8N_BASIC_AUTH_PASSWORD=your_password

   # Host settings – for local testing, set this to your VM's external IP; for production, use your domain name
   export N8N_HOST=<YOUR_DOMAIN_OR_IP>

   # The port on which n8n should listen (default is 5678)
   export N8N_PORT=5678

   # Protocol to use – use "http" for now (upgrade to "https" when TLS is set up)
   export N8N_PROTOCOL=http

   # Set your timezone
   export GENERIC_TIMEZONE=Europe/Berlin
   ```

3. **Reload Your Profile:**
   ```bash
   source ~/.bashrc
   ```

### Option B: Use a .env File

1. **Create a .env File:**
   - Navigate to your n8n folder (e.g., `~/.n8n`) and create a file named `.env`:
     ```bash
     nano ~/.n8n/.env
     ```

2. **Paste the Following Content:**
   ```env
   # n8n Basic Authentication
   N8N_BASIC_AUTH_ACTIVE=true
   N8N_BASIC_AUTH_USER=your_username
   N8N_BASIC_AUTH_PASSWORD=your_password

   # Host and Port Settings
   # For local testing, set N8N_HOST to your VM's external IP.
   N8N_HOST=<YOUR_DOMAIN_OR_IP>
   N8N_PORT=5678
   N8N_PROTOCOL=http

   # Set your timezone
   GENERIC_TIMEZONE=Europe/Berlin

   # Disable secure cookie if using HTTP (for testing only)
   N8N_SECURE_COOKIE=false
   ```

3. **Save and Close the File:**  
   In nano, press `CTRL + O` to save, then `CTRL + X` to exit.

---

## Step 6: Start n8n

You have two options to start n8n:

### Option A: Direct Command (for Testing)

Simply run:
```bash
n8n
```
n8n will start and listen on port 5678. You should see output indicating “n8n ready on 0.0.0.0, port 5678.”

### Option B: Using PM2 for Production

1. **Install PM2 Globally:**
   ```bash
   sudo npm install -g pm2
   ```

2. **Create a PM2 Ecosystem File:**
   - In your `~/.n8n` directory, create a file named `ecosystem.config.js`:
     ```bash
     nano ~/.n8n/ecosystem.config.js
     ```

3. **Paste the Following Content:**
   ```javascript
   module.exports = {
     apps: [
       {
         name: 'n8n',
         // Use the global 'n8n' command. (You can also specify the full path, e.g., '/usr/local/bin/n8n'.)
         script: 'n8n',
         // Load environment variables from the .env file located in the n8n folder.
         env_file: '/home/your_username/.n8n/.env',
         watch: false
       }
     ],

     deploy: {
       production: {
         user: 'SSH_USERNAME',
         host: 'SSH_HOSTMACHINE',
         ref: 'origin/master',
         repo: 'GIT_REPOSITORY',
         path: 'DESTINATION_PATH',
         'pre-deploy-local': '',
         'post-deploy': 'npm install && pm2 reload ecosystem.config.js --env production',
         'pre-setup': ''
       }
     }
   };
   ```
   **Important:**  
   - Replace `/home/your_username` with your actual home directory path (e.g., `/home/workablesolns` if that’s your username).  
   - Update the deployment section with your actual SSH and repository details if you plan to use PM2's deployment features.

4. **Start n8n with PM2:**
   ```bash
   pm2 start ecosystem.config.js
   ```

5. **Set PM2 to Restart on Reboot:**
   ```bash
   pm2 startup
   pm2 save
   ```

6. **Verify That n8n Is Running:**
   ```bash
   pm2 status
   ```
   You should see your n8n process listed with a status of `online`.

---

## Step 7: Access n8n

- **Directly:**  
  Open your browser and navigate to `http://<YOUR_DOMAIN_OR_IP>:5678`.

- **Using a Reverse Proxy (Optional):**  
  You may set up a reverse proxy (like Nginx or Caddy) to forward standard HTTP/HTTPS ports (80/443) to n8n’s port (5678). This step is optional and recommended for production use.

---

## Summary

1. **Create a GCP VM instance** and note the external IP.
2. **SSH into your VM** and update your system.
3. **Install Node.js and npm.**
4. **Install n8n globally using npm** and optionally create the `~/.n8n` folder.
5. **Configure environment variables** either by exporting them in your shell or using a `.env` file in your `~/.n8n` directory.
6. **Start n8n** directly for testing or with PM2 for a production-ready, continuously running setup.
7. **Access your n8n instance** via your browser.

By following these detailed steps, even someone with minimal technical knowledge should be able to set up n8n on a GCP VM using an npm-based installation. Feel free to update any placeholders with your actual values before uploading this file to GitHub.

---

Happy automating!
```
