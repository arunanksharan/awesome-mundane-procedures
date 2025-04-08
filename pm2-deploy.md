# Deploying FastAPI with Poetry and PM2 on GCP Ubuntu VM

This guide provides steps to set up your Poetry-managed FastAPI application on an Ubuntu VM in Google Cloud Platform (GCP), using PM2 as the process manager.

**Prerequisites:**

1.  A GCP Account and Project.
2.  An Ubuntu VM instance running in GCP. Note its **External IP address**.
3.  SSH access configured for your VM (e.g., via GCP key management).
4.  Your FastAPI project code available locally, managed with Poetry (`pyproject.toml` and `poetry.lock`).
5.  Project code ideally available in a Git repository (e.g., GitHub) for easy transfer.
6.  You will be running commands as a **non-root user** with `sudo` privileges for system-level installations. Replace `<username>` with your actual Linux username on the VM throughout this guide.

---

## Step 1: Connect to your GCP VM via SSH

Connect to your VM using its external IP address.

```bash
ssh <username>@<external_ip>
# Or, if using gcloud CLI:
# gcloud compute ssh <your-vm-instance-name> --zone <your-vm-zone>
```

````

---

## Step 2: Update System Packages

Ensure your system's package list and installed packages are up-to-date.

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 3: Install Prerequisites (Python, Poetry, Node.js, PM2)

**A. Install Python and Pip:**
(Often pre-installed, but good to ensure)

```bash
sudo apt install python3 python3-pip python3-venv -y
python3 --version
pip3 --version
```

**B. Install Poetry:**
Use the official installer script (run as your non-root user).

```bash
curl -sSL [https://install.python-poetry.org](https://install.python-poetry.org) | python3 -
```

Add Poetry to your PATH. The installer usually provides the command, but it's typically:

```bash
# Add to current session (replace <username>)
export PATH="/home/<username>/.local/bin:$PATH"

# Add permanently to your shell profile (replace <username>)
echo 'export PATH="/home/<username>/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify installation
poetry --version
```

**C. Install Node.js and npm:**
PM2 is a Node.js application, so Node.js and npm (Node Package Manager) are required. We'll install a recent LTS (Long-Term Support) version using NodeSource repositories.

```bash
# Install dependencies for adding repositories
sudo apt install -y ca-certificates curl gnupg

# Add NodeSource GPG key
curl -fsSL [https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key](https://www.google.com/search?q=https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key) | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

# Add NodeSource repository (Check nodesource.com for the latest LTS version, e.g., 20, 22)
NODE_MAJOR=20 # Or the current LTS version
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] [https://deb.nodesource.com/node_$NODE_MAJOR.x](https://www.google.com/search?q=https://deb.nodesource.com/node_%24NODE_MAJOR.x) nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list

# Update package list and install Node.js
sudo apt update
sudo apt install nodejs -y

# Verify installation
node -v
npm -v
```

**D. Install PM2:**
Install PM2 globally using npm.

```bash
sudo npm install pm2@latest -g

# Verify installation
pm2 --version
```

---

## Step 4: Get Your Project Code onto the VM

Transfer your FastAPI project files to the VM.

**Option A: Git Clone (Recommended)**

```bash
# Install git if needed
sudo apt install git -y

# Clone your repository
git clone <your-repository-url>
cd <your-project-directory-name> # Navigate into your project
```

**Option B: Secure Copy (`scp`)**
From your _local machine's_ terminal:

```bash
# Syntax: scp -r /path/to/local/project <username>@<external_ip>:/path/on/vm
scp -r /path/to/your/fastapi_project <username>@<external_ip>:/home/<username>/
```

Then, back in your SSH session on the VM, navigate into the copied directory:

```bash
cd /home/<username>/fastapi_project # Adjust path as needed
```

---

## Step 5: Install Python Dependencies with Poetry

Make sure you are inside your project directory on the VM. Install the dependencies defined in `pyproject.toml` / `poetry.lock`.

```bash
# Ensure you are in your project directory
# pwd

# Install dependencies (without dev dependencies)
poetry install --no-dev
```

Poetry will create and manage a virtual environment for your project.

---

## Step 6: Find the Full Path to Poetry

PM2 will need the absolute path to the `poetry` executable. Find it using `which`:

```bash
which poetry
# Example output: /home/<username>/.local/bin/poetry
# Copy this path - you'll need it soon.
```

---

## Step 7: Configure Firewall Rules in GCP

Allow incoming traffic to the port your FastAPI application will run on (e.g., 8000).

1.  Go to the GCP Console -> VPC Network -> Firewall.
2.  Click **Create Firewall Rule**.
3.  Configure the rule:
    - **Name:** `allow-fastapi-8000` (or similar)
    - **Network:** Your VM's VPC network (usually `default`)
    - **Direction:** Ingress
    - **Action:** Allow
    - **Targets:** `Specified target tags` -> enter a tag like `fastapi-server`.
    - **Source filter:** `IP ranges` -> `0.0.0.0/0` (allow all IPs - restrict if needed).
    - **Protocols and ports:** `TCP` -> enter your port (e.g., `8000`).
4.  Click **Create**.
5.  **Add the Network Tag to your VM:** Go to Compute Engine -> VM Instances -> Click your VM name -> Edit -> Network tags -> Add the tag (`fastapi-server`) -> Save.

---

## Step 8: Create a PM2 Ecosystem File

Using a PM2 ecosystem file is the recommended way to configure your application. Create a file named `ecosystem.config.js` in your project's root directory.

```bash
cd /path/to/your/project/ # Ensure you are in the project directory
nano ecosystem.config.js
```

Paste the following content into the file, **carefully replacing the placeholders**:

```javascript
module.exports = {
  apps: [
    {
      name: 'fastapi-app', // A name for your application in PM2
      script: '/home/<username>/.local/bin/poetry', // *** Full path to poetry executable (from Step 6) ***
      args: 'run uvicorn main:app --host 0.0.0.0 --port 8000', // Command poetry runs
      cwd: '/path/to/your/project/', // *** Absolute path to your project directory ***
      interpreter: 'none', // Important: Tells PM2 not to use Node.js to run the script
      env: {
        // Optional: Define environment variables your app needs
        // "DATABASE_URL": "your_db_connection_string",
        // "SECRET_KEY": "your_secret"
      },
    },
  ],
};
```

**Explanation of Placeholders:**

- `name`: How your app will be identified in PM2 commands (`pm2 list`, `pm2 logs fastapi-app`, etc.).
- `script`: The **full, absolute path** to the `poetry` executable you found in Step 6.
- `args`: The arguments passed to `poetry`. This includes `run uvicorn`, your app location (`main:app` - **adjust if needed** based on your file/object names), the host (`0.0.0.0`), and the port (`8000` - **adjust if needed** and ensure it matches your firewall rule).
- `cwd`: The **full, absolute path** to the root directory of your FastAPI project (where `pyproject.toml` resides).
- `interpreter: "none"`: Prevents PM2 from trying to run `poetry` with Node.js.
- `env`: A block to define any environment variables your application requires.

Save and close the file (`Ctrl+X`, `Y`, `Enter` in `nano`).

---

## Step 9: Start the Application with PM2

Now, use PM2 to start your application using the configuration file:

```bash
pm2 start ecosystem.config.js
```

PM2 will daemonize your application. Check its status:

```bash
pm2 list
# or
pm2 status
```

You should see `fastapi-app` (or the name you chose) with a status like `online`.

---

## Step 10: Configure PM2 to Start on System Boot

To ensure your application restarts automatically if the server reboots:

1.  **Generate the startup script command:**

    ```bash
    pm2 startup
    ```

    This command will analyze your system (detecting `systemd`) and output _another command_ that you need to run with `sudo`. It will look something like this ( **copy the exact command from YOUR terminal** ):
    `sudo env PATH=$PATH:/usr/bin /home/<username>/.npm-global/bin/pm2 startup systemd -u <username> --hp /home/<username>`

2.  **Run the command provided by PM2:**
    Paste the command you copied from the `pm2 startup` output and run it.

3.  **Save the current PM2 process list:**
    ```bash
    pm2 save
    ```
    This saves the list of apps currently running so the startup script knows what to resurrect on boot.

---

## Step 11: Final Verification and PM2 Usage

1.  **Check Logs:** View your application's logs:

    ```bash
    pm2 logs fastapi-app  # Use the app name you defined
    # Press Ctrl+C to exit logs
    ```

2.  **Test Externally:** Access your application from your browser or `curl` using the VM's external IP and the configured port:

    ```bash
    curl http://<external_ip>:8000/
    # Try accessing specific endpoints like /docs
    curl http://<external_ip>:8000/docs
    ```

    If you get a response, everything is working!

3.  **Common PM2 Commands:**
    - `pm2 list` or `pm2 status`: Show running processes.
    - `pm2 restart fastapi-app`: Restart your app.
    - `pm2 stop fastapi-app`: Stop your app.
    - `pm2 delete fastapi-app`: Stop and remove the app from PM2's list.
    - `pm2 monit`: Monitor CPU/Memory usage in real-time.

---

You have now successfully deployed your FastAPI application using Poetry on a GCP Ubuntu VM, managed by PM2! Your application will run in the background and restart automatically on boot or if it crashes.

---
```
```

---

````
