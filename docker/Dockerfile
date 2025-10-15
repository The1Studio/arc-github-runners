# Custom GitHub Actions Runner with HTTPS APT sources
# Based on official summerwind/actions-runner image
# Pin specific version to prevent supply chain attacks and ensure reproducibility
FROM summerwind/actions-runner:v2.328.0-ubuntu-22.04

# Switch to root to modify system files
USER root

# Fix APT sources to use HTTPS instead of HTTP (port 80 blocked in k8s)
RUN sed -i 's|http://archive.ubuntu.com|https://archive.ubuntu.com|g' /etc/apt/sources.list && \
    sed -i 's|http://security.ubuntu.com|https://security.ubuntu.com|g' /etc/apt/sources.list

# Disable PPAs that don't support HTTPS
RUN find /etc/apt/sources.list.d/ -type f -name "*.list" \
    -exec sed -i 's|^deb http://ppa.launchpad.net|# deb http://ppa.launchpad.net|g' {} \;

# Update package lists and clean up
RUN apt-get update && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Switch back to runner user (non-root)
USER runner

# Label for tracking
LABEL maintainer="tuha263" \
      description="GitHub Actions Runner with HTTPS APT sources" \
      version="1.1" \
      base-image="summerwind/actions-runner:v2.328.0-ubuntu-22.04" \
      issue="port-80-http-blocked"
