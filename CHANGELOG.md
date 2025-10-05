# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-05-10

### Added
- Initial release of the Infrastructure Repository
- Docker Compose configurations for multiple services:
  - Nginx reverse proxy with SSL support
  - Jitsi video conferencing platform
  - WordPress content management system
  - Nextcloud file sharing and collaboration platform
  - Watchtower for automatic container updates
  - Monitoring stack
  - Jump host configuration
  - Boinc service to provide compute resources to science
- Comprehensive documentation in README.md
- Security features:
  - Fail2ban integration for Nextcloud
  - SSL/TLS certificate management with Certbot
  - Firewall configuration guidelines
- Service template for easy addition of new services
- GitHub workflow configurations
- YAML linting configuration
- Git configuration and hooks setup
- License file (Apache License)
- Contributing guidelines

### Known Issues
- Nginx SSL configuration requires manual certificate setup before enabling SSL
- Some services may require additional configuration for production use
- Beta status for certain components, particularly the nginx-example

### Technical Details
- Tested on Ubuntu 24.04
- Requires Docker Compose v2.33.1 or later
- Includes comprehensive setup instructions for each service
- Provides environment variable templates for service configuration
