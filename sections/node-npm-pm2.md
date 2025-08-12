# Node & npm & pm2

For the most current Node.js version, use [the official Node.js website](https://nodejs.org/en/download/current).
**Avoid installing Node.js via `apt install node` as it is usually outdated.**

```bash
# Download and install nvm (Node Version Manager) to easily manage Node.js versions
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# Load nvm without restarting the shell (for current session)
\. "$HOME/.nvm/nvm.sh"

# Download and install the latest Node.js (replace 24 with your desired version if preferred)
nvm install 24

# Verify Node.js installation
node -v         # Should print something like "v24.4.1"
nvm current     # Should also reflect the installed version

# Check npm (Node Package Manager) version
npm -v          # Should print a current version, e.g. "11.4.2"

# Install pm2 globally (process manager for Node apps)
npm install -g pm2
```