# MySQL SSH Tunnel Manager

A robust bash script for managing persistent SSH tunnels to remote MySQL servers using autossh. Perfect for connecting to databases behind VPNs or firewalls where direct connections aren't possible.

## Features

- **Process Management**: PID file tracking with automatic cleanup of stale processes
- **Robust Restart Logic**: Properly waits for tunnels to terminate before restarting
- **Comprehensive Logging**: Central manager log plus individual tunnel logs
- **Configuration File**: Simple pipe-delimited format for managing multiple tunnels
- **Auto-Bootstrap**: Automatically creates required directories with sudo when needed
- **VPN-Resilient**: autossh automatically reconnects when network connections drop
- **Individual Control**: Start, stop, or restart specific tunnels or all at once

## Quick Start

1. Clone the repository:
```bash
git clone <your-repo-url>
cd mysql-tunnel-manager
```

2. Make the script executable:
```bash
chmod +x tunnels.sh
```

3. Edit `tunnels.conf` to add your database servers:
```
# Format: name|local_port|remote_host|remote_port|ssh_host|ssh_options
gce-db-3|33303|localhost|3306|gce-db-3|
gce-db-4|33304|localhost|3306|gce-db-4|
```

4. Start all tunnels:
```bash
./tunnels.sh start
```

The script will automatically create `/var/run/autossh` and `/var/log/autossh` with proper permissions (prompting for sudo if needed).

## Usage

### Basic Commands

Start all tunnels:
```bash
./tunnels.sh start
```

Start a specific tunnel:
```bash
./tunnels.sh start gce-db-3
```

Stop all tunnels:
```bash
./tunnels.sh stop
```

Stop a specific tunnel:
```bash
./tunnels.sh stop gce-db-4
```

Restart all tunnels:
```bash
./tunnels.sh restart
```

Restart a specific tunnel:
```bash
./tunnels.sh restart gce-db-3
```

Check tunnel status:
```bash
./tunnels.sh status
```

### Example Output

```bash
$ ./tunnels.sh status
[2026-01-07 10:15:30] Checking tunnel status...
✓ gce-db-3: RUNNING (PID: 1514250, Port: 33303)
✓ gce-db-4: RUNNING (PID: 1514289, Port: 33304)
```

## Configuration

### Tunnel Configuration File

Edit `tunnels.conf` to define your tunnels. Each line represents one tunnel:

```
name|local_port|remote_host|remote_port|ssh_host|ssh_options
```

**Fields:**
- `name`: Unique identifier for the tunnel
- `local_port`: Local port to bind (e.g., 33303)
- `remote_host`: Remote MySQL host (usually 'localhost' on the SSH server)
- `remote_port`: Remote MySQL port (default: 3306)
- `ssh_host`: SSH server hostname or alias from ~/.ssh/config
- `ssh_options`: Additional SSH options (optional)

**Example configurations:**

```conf
# Basic tunnel
prod-db|33306|localhost|3306|db-server|

# Tunnel with custom SSH options
staging-db|33307|localhost|3306|staging-host|-i ~/.ssh/staging_key -p 2222

# Tunnel to non-standard MySQL port
legacy-db|33308|10.0.1.50|3307|bastion-host|
```

### SSH Configuration

For best results, set up SSH host aliases in `~/.ssh/config`:

```
Host gce-db-3
    HostName 203.0.113.10
    User dbadmin
    IdentityFile ~/.ssh/gce-db-key
    ServerAliveInterval 30
    ServerAliveCountMax 3

Host gce-db-4
    HostName 203.0.113.11
    User dbadmin
    IdentityFile ~/.ssh/gce-db-key
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

### Environment Variables

You can customize locations using environment variables:

```bash
# Use custom configuration file
CONFIG_FILE=~/my-tunnels.conf ./tunnels.sh start

# Use custom PID directory
PID_DIR=~/tunnels/pids ./tunnels.sh start

# Use custom log directory
LOG_DIR=~/tunnels/logs ./tunnels.sh start
```

**Defaults:**
- `CONFIG_FILE`: `./tunnels.conf` (in script directory)
- `PID_DIR`: `/var/run/autossh`
- `LOG_DIR`: `/var/log/autossh`

## Connecting to Your Databases

Once tunnels are running, connect to your databases via localhost:

```bash
# Connect to gce-db-3 (port 33303)
mysql -h 127.0.0.1 -P 33303 -u dbuser -p

# Connect to gce-db-4 (port 33304)
mysql -h 127.0.0.1 -P 33304 -u dbuser -p
```

Or use MySQL Workbench, DBeaver, or any other client with:
- **Host**: 127.0.0.1
- **Port**: 33303 (or your configured port)
- **Username/Password**: Your remote database credentials

## Directory Structure

```
mysql-tunnel-manager/
├── tunnels.sh          # Main script
├── tunnels.conf        # Tunnel configuration
├── README.md           # This file
├── /var/run/autossh/   # PID files (auto-created)
│   ├── gce-db-3.pid
│   └── gce-db-4.pid
└── /var/log/autossh/   # Log files (auto-created)
    ├── tunnel-manager.log
    ├── gce-db-3.log
    └── gce-db-4.log
```

## Troubleshooting

### Tunnel won't start

Check the tunnel-specific log:
```bash
tail -f /var/log/autossh/gce-db-3.log
```

Common issues:
- SSH host not reachable (VPN down?)
- SSH authentication failure (check keys)
- Port already in use locally
- Remote port not accessible

### Check main manager log

```bash
tail -f /var/log/autossh/tunnel-manager.log
```

### Verify SSH connection manually

```bash
ssh -v gce-db-3
```

### Check if port is already in use

```bash
lsof -i :33303
```

### Force cleanup

If you have stale processes:
```bash
pkill -f autossh
rm /var/run/autossh/*.pid
./tunnels.sh start
```

## Requirements

- `autossh` - Install with:
  - Ubuntu/Debian: `sudo apt-get install autossh`
  - CentOS/RHEL: `sudo yum install autossh`
  - macOS: `brew install autossh`
- `bash` 4.0 or higher
- SSH access to remote servers
- `sudo` access for initial directory setup (if using default locations)

## How It Works

1. **autossh** maintains persistent SSH tunnels with automatic reconnection
2. **Port forwarding** maps local ports to remote MySQL servers
3. **PID files** track running processes and prevent duplicates
4. **Health checks** use SSH ServerAliveInterval to detect dead connections
5. **Graceful shutdown** ensures ports are freed before restart

The script uses autossh's built-in monitoring (ServerAliveInterval) rather than the legacy `-M` monitor port, which is more reliable and doesn't require extra port allocation.

## systemd Integration (Optional)

To run tunnels as a system service, create `/etc/systemd/system/mysql-tunnels.service`:

```ini
[Unit]
Description=MySQL SSH Tunnel Manager
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
User=your-username
WorkingDirectory=/path/to/mysql-tunnel-manager
ExecStart=/path/to/mysql-tunnel-manager/tunnels.sh start
ExecStop=/path/to/mysql-tunnel-manager/tunnels.sh stop
ExecReload=/path/to/mysql-tunnel-manager/tunnels.sh restart
RemainAfterExit=yes
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable mysql-tunnels
sudo systemctl start mysql-tunnels
sudo systemctl status mysql-tunnels
```

## License

MIT License - feel free to use and modify as needed.

## Contributing

Pull requests welcome! Please ensure:
- Code follows existing style
- Changes are tested with multiple tunnels
- README is updated for new features

## Author

Created for managing ESP database infrastructure and job board platforms with reliable tunnel connections.