# FOSSEE System Administrator Task

> This repository contains my submission for the FOSSEE Semester Long Internship - Autumn 2025.

## Table of Contents

- [Project Overview](#project-overview)
- [Live Environment Access](#live-environment-access)
- [Documentation Files](#documentation-files)

## Project Overview

This repository contains the complete documentation and resources for setting up secure, production ready Digital Ocean droplet running Rocky Linux 10, with Keycloak integrated for Single Sign-On (SSO). The server is configured to run Keycloak for Single Sign-On (SSO) integration with three applications: Drupal 11, a Django project, and a generic PHP application.

## Live Environment Access

- **Droplet IP:** 139.59.68.14
- **Keycloak Admin Console:** https://keycloak.swaruph.tech/
- **Drupal Application:** https://drupal.swaruph.tech/
- **Django Application:** https://django.swaruph.tech/
- **PHP Application:** https://phpapp.swaruph.tech/

## Documentation Files

| File                                                   | Description                                                                                       |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| [01-server-setup.md](./01-server-setup.md)             | Digital Ocean droplet creation and server setup steps including firewall and package installation |
| [02-keycloak-setup.md](./02-keycloak-setup.md)         | Keycloak installation, MariaDB configuration and Apache reverse proxy with SSL                    |
| [03-drupal-integration.md](./03-drupal-integration.md) | Drupal 11 installation and configuration with Keycloak SSO integration                            |
| [04-django-integration.md](./04-django-integration.md) | Django application setup and integration with Keycloak SSO                                        |
| [05-php-integration.md](./05-php-integration.md)       | PHP application setup and integration with Keycloak SSO                                           |
