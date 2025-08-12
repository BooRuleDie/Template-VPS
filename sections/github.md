# GitHub Setup

```bash
# Generate a new SSH key pair (replace the email with yours)
ssh-keygen -t ed25519 -C "your_email@example.com"   # Press Enter through prompts

# Copy the content of your public key to clipboard
cat ~/.ssh/id_ed25519.pub
```

1. Log in to your GitHub account.
2. Go to 'Settings'.
3. Navigate to `SSH and GPG keys`.
4. Click `New SSH Key`.
5. Give it a descriptive name and paste the content you just copied.
6. Test your SSH connection with the following command:

```bash
ssh -T git@github.com
# Hi BooRuleDie! You've successfully authenticated, but GitHub does not provide shell access.
```