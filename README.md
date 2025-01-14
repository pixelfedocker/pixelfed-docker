# Pixelfed Self-Hosting Guide

A comprehensive guide for self-hosting Pixelfed using Docker.

## Important Notice
This is an unofficial guide created by community contributors and AI assistance. Always refer to [official Pixelfed documentation](https://docs.pixelfed.org/) for critical deployments.

## Quick Start

### Prerequisites
- Linux server (Ubuntu 24.04 LTS recommended)
- 2GB RAM minimum (4GB recommended)
- 20GB storage
- Domain name
- Basic terminal knowledge

### Copy this git
```bash
git clone https://github.com/pixelfedocker/pixelfed-docker
cd pixelfed-docker
./scripts/setup.sh
```

## Documentation

1. [Core Setup](docs/01-core-setup.md)
   - System Requirements
   - Docker Installation
   - Basic Configuration

2. [Network & SSL](docs/02-network-ssl.md)
   - DNS Configuration
   - SSL Certificates
   - Nginx Setup

3. [Maintenance](docs/03-maintenance.md)
   - Backup & Recovery
   - Updates
   - Monitoring

4. [Email Setup](docs/04-email-setup.md)
   - Mail Server Configuration
   - Integration
   - Testing

5. [Security](docs/05-security.md)
   - Hardening Guide
   - Best Practices
   - Monitoring

## Support & Community

- [Discord](https://discord.gg/pixelfed)
