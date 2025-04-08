Okay, here's a comprehensive step-by-step guide to deploying your FastAPI project using Poetry and PM2 on an Ubuntu server. This guide includes setting up the server, installing dependencies, configuring the process manager (PM2), and optionally setting up Nginx as a reverse proxy.

**Assumptions:**

- You have SSH access to a fresh Ubuntu server (e.g., Ubuntu 20.04, 22.04).
- You have root or sudo privileges.
- Your FastAPI application code is ready (e.g., in a Git repository).
- Your project uses Poetry for dependency management (`pyproject.toml` file exists).
- Your FastAPI app object is accessible, e.g., `app.main:app` (where `main.py` is in the `app` directory, and `app` is the FastAPI instance). Adjust this path as needed.

---

**Step 1: Server Preparation**

First, connect to your server via SSH and update the system packages. Install Python, pip, tools for building packages, and Nginx (we'll configure Nginx later).

```bash
# Update package lists and upgrade existing packages
sudo apt update && sudo apt upgrade -y

# Install Python3, pip, venv, Nginx, curl, and build tools
sudo apt install -y python3 python3-pip python3-venv build-essential libssl-dev libffi-dev python3-dev nginx curl git
```

---

**Step 2: Install Poetry**

Install Poetry using the official installer.

```bash
# Download and execute the official Poetry installer script
curl -sSL https://install.python-poetry.org | python3 -

# Add Poetry to your PATH environment variable
# The command below might differ slightly based on the installer's output. Follow the instruction shown after installation.
# Typically, it looks like this (replace <username> with your actual username):
export PATH="/home/<username>/.local/bin:$PATH"

# Add the export line to your shell profile for persistence (e.g., ~/.bashrc or ~/.profile)
# Example for bashrc:
echo 'export PATH="/home/<username>/.local/bin:$PATH"' >> ~/.bashrc

# Apply the changes to the current session
source ~/.bashrc

# Verify installation
poetry --version
```

---

**Step 3: Get Your Project Code**

Clone your project repository or copy your project files onto the server.

```bash
# Clone your project (replace <your-repository-url>)
git clone <your-repository-url>

# Navigate into your project directory (replace <your-project-directory>)
cd <your-project-directory>
```

---

**Step 4: Set Up Project Environment with Poetry**

Install your project dependencies using Poetry. It's good practice to configure Poetry to create the virtual environment inside your project directory (`.venv`) for easier reference.

```bash
# (Optional but Recommended) Configure Poetry to create the virtualenv in the project folder
poetry config virtualenvs.in-project true

# Install dependencies (use --no-dev for production to skip development dependencies)
poetry install --no-dev
```

---

**Step 5: Install ASGI Server (Uvicorn)**

Ensure your ASGI server (like Uvicorn) is listed as a dependency in your `pyproject.toml`. If not, add it:

```bash
# Add uvicorn with standard performance extras (like uvloop)
poetry add uvicorn[standard]

# Re-run install if you added it manually to pyproject.toml
# poetry install --no-dev
```

---

**Step 6: Test Application Locally (Optional but Recommended)**

Before configuring PM2, test if Uvicorn can run your app directly from the Poetry environment.

```bash
# Find the path to the virtual environment's Python interpreter/scripts
# This command will output the path, note it down.
poetry env info --path
# Let's call the output <venv_path> (e.g., /home/user/myproject/.venv)

# Run Uvicorn using the interpreter in the virtual environment
# Replace <venv_path>, app.main:app, and port 8000 if needed
<venv_path>/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000

# Access http://<your_server_ip>:8000 in your browser to test.
# Stop the server with Ctrl+C once tested.
```

---

**Step 7: Install Node.js and PM2**

PM2 is a Node.js application, so you need to install Node.js and npm first. We'll use NodeSource repositories for a recent version.

```bash
# Install Node.js (e.g., LTS version 20 - check nodesource.com for current LTS)
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

NODE_MAJOR=20 # Or current LTS version
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list

sudo apt update
sudo apt install nodejs -y

# Verify Node.js and npm installation
node -v
npm -v

# Install PM2 globally using npm
sudo npm install pm2@latest -g

# Verify PM2 installation
pm2 --version
```

---

**Step 8: Configure PM2 to Run Your FastAPI App**

The best way to configure PM2 for complex applications is using an `ecosystem.config.js` file. This allows you to specify the interpreter path (from your Poetry venv), arguments, working directory, and environment variables.

1.  **Get Required Paths:**

    - Virtual Environment Path (`<venv_path>`): Run `poetry env info --path` again in your project directory.
    - Project Directory Path (`<project_path>`): Run `pwd` in your project directory.

2.  **Create `ecosystem.config.js`:**
    In your project's root directory, create a file named `ecosystem.config.js`:

    ```bash
    nano ecosystem.config.js
    ```

3.  **Add Configuration:** Paste and **edit** the following content, replacing placeholders:

    ```javascript
    module.exports = {
      apps: [
        {
          name: 'fastapi-app', // Your application's name in PM2
          script: '<venv_path>/bin/uvicorn', // Absolute path to uvicorn in your .venv
          args: 'app.main:app --host 127.0.0.1 --port 8000', // Arguments for Uvicorn (use 127.0.0.1 if using Nginx)
          // Replace app.main:app and port 8000 if needed
          // interpreter: "<venv_path>/bin/python", // Usually not needed if 'script' points to venv uvicorn
          cwd: '<project_path>', // Absolute path to your project directory
          env: {
            // Optional: Environment variables your app needs
            // "DATABASE_URL": "your_db_connection_string",
            // "SECRET_KEY": "your_secret"
          },
        },
      ],
    };
    ```

    - Replace `<venv_path>` with the actual path from `poetry env info --path`.
    - Replace `<project_path>` with the actual path from `pwd`.
    - Replace `"fastapi-app"` with your desired app name.
    - Replace `"app.main:app --host 127.0.0.1 --port 8000"` with your actual app location, host (use `127.0.0.1` if proxying with Nginx, `0.0.0.0` otherwise), and desired port.
    - Add any necessary environment variables under `env`.

4.  **Start the Application with PM2:**

    ```bash
    pm2 start ecosystem.config.js
    ```

5.  **Check Status:**
    ```bash
    pm2 list
    # or
    pm2 status
    # Check logs
    pm2 logs fastapi-app # Use the name you defined in the ecosystem file
    ```

---

**Step 9: Set Up PM2 Startup Script**

Configure PM2 to automatically restart your application if the server reboots.

```bash
# Generate the startup command
pm2 startup

# Follow the instructions printed by the command. It will likely give you a command
# like 'sudo env PATH=$PATH:/usr/bin /home/<username>/.local/bin/pm2 startup ...'
# Copy and run that specific command.

# Save the current list of processes managed by PM2
pm2 save
```

---

**Step 10: Configure Nginx as a Reverse Proxy (Recommended)**

Using Nginx allows you to handle incoming HTTP/HTTPS requests, manage TLS/SSL certificates, serve static files efficiently, and forward requests to your FastAPI application running via PM2/Uvicorn.

1.  **Create Nginx Configuration File:**

    ```bash
    sudo nano /etc/nginx/sites-available/fastapi_app
    ```

2.  **Add Server Block:** Paste and **edit** the following basic configuration:

    ```nginx
    server {
        listen 80;
        server_name your_domain_or_server_ip; # Replace with your actual domain or IP

        location / {
            # Forward requests to the Uvicorn process managed by PM2
            proxy_pass http://127.0.0.1:8000; # Must match the host and port Uvicorn is listening on (Step 8)

            # Set headers to pass information to your FastAPI app
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme; # Important for detecting https
        }

        # Optional: Add locations for static files if needed
        # location /static {
        #     alias /path/to/your/project/static;
        # }
    }
    ```

    - Replace `your_domain_or_server_ip` with your server's public IP address or domain name.
    - Ensure `proxy_pass http://127.0.0.1:8000` points to the correct host and port where Uvicorn is listening (as configured in `ecosystem.config.js`).

3.  **Enable the Site:** Create a symbolic link to the `sites-enabled` directory.

    ```bash
    sudo ln -s /etc/nginx/sites-available/fastapi_app /etc/nginx/sites-enabled/

    # Optional: Remove default Nginx welcome page link if it exists
    sudo rm /etc/nginx/sites-enabled/default
    ```

4.  **Test Nginx Configuration:**

    ```bash
    sudo nginx -t
    ```

    If it reports syntax is OK, proceed.

5.  **Restart Nginx:**

    ```bash
    sudo systemctl restart nginx
    ```

6.  **(Highly Recommended) Add HTTPS with Let's Encrypt:**
    If you have a domain name pointing to your server, use Certbot to easily set up free SSL certificates.

    ```bash
    # Install Certbot and the Nginx plugin
    sudo apt install certbot python3-certbot-nginx -y

    # Obtain and install certificate (follow prompts)
    # Replace your_domain_or_server_ip with your domain
    sudo certbot --nginx -d your_domain_or_server_ip
    ```

    Certbot will automatically modify your Nginx configuration for HTTPS and set up auto-renewal.

---

**Step 11: Configure Firewall (UFW)**

Allow traffic on the necessary ports (SSH, HTTP, HTTPS).

```bash
# Check firewall status
sudo ufw status

# If inactive, enable it (MAKE SURE TO ALLOW SSH FIRST if connected via SSH)
sudo ufw allow OpenSSH # Or 'sudo ufw allow 22'
sudo ufw enable

# Allow Nginx traffic (handles port 80 and 443)
sudo ufw allow 'Nginx Full'

# Verify rules
sudo ufw status
```

---

**Step 12: Final Checks**

- Access your application using your server's IP address or domain name (e.g., `http://your_domain_or_server_ip` or `https://your_domain_or_server_ip` if you set up HTTPS).
- Verify PM2 is running your app: `pm2 list`.
- Monitor logs if needed: `pm2 logs fastapi-app`.
- Check Nginx status/logs if you encounter connection issues: `sudo systemctl status nginx`, `/var/log/nginx/error.log`.

Your FastAPI application should now be deployed and managed by PM2, with Poetry handling dependencies and Nginx acting as a reverse proxy.
