# Setting up SSH Key Authentication between Ubuntu Server and GitHub

This guide explains how to generate an SSH key on your Ubuntu server and add it to your GitHub account for secure, passwordless Git operations.

**Prerequisites:**

- You are logged into your Ubuntu server via SSH.
- You have a GitHub account.

---

## Step 1: Check for Existing SSH Keys

First, check if you already have an SSH key pair generated on the server. Common key filenames include `id_rsa`, `id_ed25519`, `id_ecdsa`.

```bash
ls -al ~/.ssh
```

Look for files named like `id_*.pub` (public key) and `id_*` (private key).

- If you see existing keys (like `id_ed25519` and `id_ed25519.pub`) and you want to use them, you can skip Step 2 and proceed to Step 3 (if they have a passphrase) or Step 4.
- If you don't see suitable keys or want to create a new dedicated key, proceed to Step 2.

---

## Step 2: Generate a New SSH Key Pair

If you need to generate a new key, use the `ssh-keygen` command. Using the Ed25519 algorithm is generally recommended as it's more secure and performant than RSA.

```bash
# Replace "[email address removed]" with the email associated with your GitHub account
ssh-keygen -t ed25519 -C "[email address removed]"
```

You will be prompted for a few things:

1.  **Enter file in which to save the key (`/home/your_username/.ssh/id_ed25519`):** Press `Enter` to accept the default location. If a key file already exists, it will ask if you want to overwrite - be careful if you have existing keys you use for other purposes.
2.  **Enter passphrase (empty for no passphrase):**
    - **Recommended:** Enter a strong passphrase. This adds an extra layer of security. If someone gains access to your private key file, they still need the passphrase to use it. You might need an SSH agent (see Step 3) to avoid typing it repeatedly.
    - **Easier (Less Secure):** Press `Enter` twice to have no passphrase. This is convenient, especially for automated scripts, but less secure if your private key file is compromised.

This command will generate two files in your `~/.ssh/` directory (by default):

- `id_ed25519` (Your **private** key - KEEP THIS SECRET AND SECURE!)
- `id_ed25519.pub` (Your **public** key - This is what you share/upload)

**Set Correct Permissions (Important for Security):**
Ensure your private key is only readable by you:

```bash
chmod 600 ~/.ssh/id_ed25519
```

---

## Step 3: Add Your SSH Private Key to the ssh-agent (Optional, if using a passphrase)

If you set a passphrase in Step 2 and want to avoid typing it every time you use the key during your current session, you can add it to the `ssh-agent`.

1.  **Ensure the ssh-agent is running:**

    ```bash
    eval "$(ssh-agent -s)"
    ```

    (This should output an Agent pid)

2.  **Add your SSH private key to the agent:**
    ```bash
    ssh-add ~/.ssh/id_ed25519
    ```
    You will be prompted to enter the passphrase you created in Step 2.

_Note: The agent typically only remembers the key for your current login session._

---

## Step 4: Get Your Public Key

You need to copy the content of your **public** key file to add it to GitHub. Use `cat` to display it:

```bash
cat ~/.ssh/id_ed25519.pub
```

This will output a long string starting with `ssh-ed25519` (or `ssh-rsa` if you generated an RSA key) and ending with your email address (the comment). **Copy this entire output to your clipboard.** Make sure you copy everything, without adding extra lines or spaces.

---

## Step 5: Add the Public Key to Your GitHub Account

1.  Open your web browser and log in to your GitHub account.
2.  Click on your profile picture in the top-right corner, then click **Settings**.
3.  In the left sidebar, under "Access", click **SSH and GPG keys**.
4.  Click the **New SSH key** or **Add SSH key** button.
5.  In the **Title** field, give your key a descriptive name so you can recognize it later (e.g., "GCP FastAPI Server", "My Ubuntu VM").
6.  In the **Key** field, paste the **entire public key** you copied in Step 4.
7.  Ensure the **Key type** is set to `Authentication Key`.
8.  Click **Add SSH key**. You might be asked to confirm your GitHub password or use 2FA.

---

## Step 6: Test Your SSH Connection to GitHub

Now, back in your Ubuntu server's terminal, test if the connection works:

```bash
ssh -T git@github.com
```

You might see a message like this the first time you connect:

```
The authenticity of host 'github.com (XXX.XXX.XXX.XXX)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes` and press `Enter`.

If successful, you should see a message similar to this:

```
Hi <your-github-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

If you see this message, your SSH key is correctly set up! You can now use `git clone`, `git pull`, `git push`, etc., with repositories you have access to on GitHub using the SSH URL format (e.g., `git@github.com:username/repo.git`).

---
