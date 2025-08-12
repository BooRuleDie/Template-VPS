# Server Setup

```bash
# Update the package index and upgrade all installed packages
sudo apt update && sudo apt upgrade -y

# Create a new user (replace <username> with your desired username)
adduser <username>

# Add the new user to the sudo group to grant admin privileges
usermod -aG sudo <username>

# Switch to the new user account
su - <username>

# Set up SSH configuration for the new user
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys   # Paste your public key here for key-based authentication
chmod 600 ~/.ssh/authorized_keys

# Disable root login over SSH for enhanced security
sudo nano /etc/ssh/sshd_config   # Change 'PermitRootLogin yes' to 'PermitRootLogin no'
sudo systemctl restart ssh       # Restart SSH to apply changes

# Install btop for system monitoring
sudo apt install btop -y

# (Optional) Update the system's timezone
timedatectl list-timezones | grep Istanbul # List all available timezones and filter for "Istanbul" to find the correct timezone string
sudo timedatectl set-timezone Europe/Istanbul # Set the system timezone to Europe/Istanbul
timedatectl # Display the current date, time, and timezone settings to verify the change

# Set up Zsh and install essentials
sudo apt install -y zsh git curl
chsh -s $(which zsh)  # Change the user's default shell to Zsh. Logout and log back in (exit and reconnect by SSH) to use Zsh.

# Install Oh My Zsh (improves Zsh experience with themes and plugins)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install Zsh plugins for autosuggestions and syntax highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
nano ~/.zshrc   # Edit the plugins line: plugins=(git) → plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
source ~/.zshrc # Reload the Zsh configuration

# (Optional) .envrc setup with direnv
sudo apt install -y direnv

# Hook direnv into your shell (Zsh)
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
source ~/.zshrc

# For Bash users, uncomment the following:
# echo 'eval "$(direnv hook bash)"' >> ~/.bashrc && source ~/.bashrc

# Allow direnv to load .envrc for this directory
direnv allow
```