# Traefik Private PKI Proof of Concept

This repository demonstrates how to use Traefik with self-signed certificates from a private PKI (Public Key Infrastructure) for serving HTTP and TCP secured connections. It shows how to set up and configure Traefik to use custom certificates for TLS termination.

## Overview

Traefik is a modern HTTP reverse proxy and load balancer that makes deploying microservices easy. This proof of concept focuses on using Traefik with a private PKI, allowing you to:

- Secure services with self-signed certificates
- Configure Traefik to trust your private CA
- Demonstrate TLS termination for both HTTP and TCP services
- Show different certificate formats and how Traefik handles them

## Prerequisites

- Docker and Docker Compose
- Basic understanding of TLS/SSL certificates
- Familiarity with Traefik configuration

## Project Structure

```
.
├── certs/                  # Certificate files (deployed to Traefik)
│   ├── any.example.tld.crt # Certificate for all subdomains
│   ├── any.example.tld.key # Private key
│   └── ca.crt              # Root CA certificate
├── easy-rsa/               # Certificate management toolkit
├── etc/
│   └── tls.yml             # Traefik TLS configuration
├── pki/                    # PKI directory (certificate database)
├── compose.yaml            # Docker Compose configuration
├── Justfile                # Helper commands
└── README.md               # This file
```

## Certificate Management

This project uses Easy-RSA to create and manage certificates. The PKI (Public Key Infrastructure) is stored in the pki directory.

Key files:

- ca.crt: The CA (Certificate Authority) certificate
- ca.key: The CA private key
- any.example.tld.crt: The wildcard certificate for *.example.tld
- any.example.tld.key: The private key for the wildcard certificate

The certificates are copied to the certs directory for deployment with Traefik.

### Inspecting Certificates
Use the following command to inspect the CA certificate:

```bash
just info_ca
```

or use OpenSSL:

```bash
openssl x509 -noout -text -in pki/ca.crt
```

## Traefik Configuration

Traefik is configured to:

1. Trust our private CA
2. Use the wildcard certificate for HTTPS services
3. Properly handle TLS termination for TCP services


```yaml
  api-gateway:
    image: traefik:v2.11
    restart: unless-stopped
    command:
      - "--api.insecure=true"
      - "--api.dashboard=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.http.tls=true"
      # stream is an arbitrary name, for representing the tcp entrypoint
      - "--entrypoints.stream.address=:1234"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.file.filename=/etc/traefik/conf.d/tls.yml"
      - "--providers.file.watch=true"
      - "--log.level=WARN"
      - "--accesslog=true"
      # - "--serversTransport.insecureSkipVerify=true"
    ports:
      - "80:80"
      - "443:443"
      - "1234:1234"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./etc/tls.yml:/etc/traefik/conf.d/tls.yml:ro
      - ./certs:/etc/certs
    labels:
      - traefik.enable=true
      
      - traefik.http.services.api-gateway.loadbalancer.server.port=8080
      - traefik.http.routers.api-gateway.rule=Host(`ingress.${DOMAIN}`)
      - traefik.http.routers.api-gateway.entrypoints=web
      - traefik.http.routers.api-gateway.middlewares=ingress-auth,redirect-web-secure

      - traefik.http.routers.api-gateway-secure.rule=Host(`ingress.${DOMAIN}`)
      - traefik.http.routers.api-gateway-secure.entrypoints=websecure
      - traefik.http.routers.api-gateway-secure.tls=true
      - traefik.http.routers.api-gateway-secure.middlewares=ingress-auth,redirect-web-secure

      - traefik.http.middlewares.ingress-auth.basicauth.users=${INGRESS_USER}:${INGRESS_ACCESS_HTPASSWD}
      - traefik.http.middlewares.redirect-web-secure.redirectscheme.scheme=https
      - traefik.http.middlewares.redirect-web-secure.redirectscheme.permanent=true
    networks:
      - internal
```

## Detailed Traefik Configuration Explanation

Traefik is configured through a combination of command-line arguments in the Docker Compose file and the dedicated TLS configuration file.

### Entrypoints

Three entrypoints are configured:
- **web**: Standard HTTP on port 80
- **websecure**: HTTPS on port 443 with TLS enabled
- **stream**: TCP on port 1234 for non-HTTP traffic

### TLS Termination

Traefik is configured to use our custom certificates for TLS termination:
- The wildcard certificate (`any.example.tld.crt`) is used for all secured services
- TLS 1.3 is set as the minimum allowed protocol version
- SNI strict mode ensures clients must specify the hostname they're connecting to

### Service Discovery

Traefik uses two configuration providers:
1. **Docker provider**: Discovers services and their routing rules from container labels
2. **File provider**: Loads TLS configuration from the `tls.yml` file

### Security Features

The configuration includes several security enhancements:
- Basic authentication for the dashboard
- HTTP to HTTPS redirects
- Content Security Policy headers
- Custom response headers for CORS

### Service Routing Examples

1. **HTTP Service (whoami)**: 
   - Accessible at `whoami.example.tld`
   - Automatically redirects HTTP to HTTPS
   - Uses middleware chains for header manipulation

2. **TCP Service (echo)**:
   - Accessible at `echo.example.tld:1234`
   - Uses SNI for routing
   - TLS encryption for the TCP connection

## Usage

### Starting the Services

```bash
docker-compose up -d
```

### Testing the Setup

Description for Linux CLI:

1. Register ca.crt as a trusted CA:

   ```bash
   sudo cp certs/ca.crt /usr/local/share/ca-certificates/
   sudo update-ca-certificates
   ```

1. Access secure HTTP services `whoami`:

   ```bash
   curl -Lv https://whoami.example.tld
   ```

1. Test TCP services:

   ```bash
   openssl s_client -connect service2.example.tld:443
   # when connected just type something and press enter
   # you should see the echo response
   ```

## Common Issues and Troubleshooting

- **Certificate not trusted**: Ensure the CA certificate is properly configured in Traefik
- **SNI routing issues**: Verify the Host rules in your Traefik configuration
- **TCP services not accessible**: Check the TCP router configuration and certificate paths
- **DNS resolution issues**: Ensure your DNS is correctly configured

## Advanced Certificate Handling

As noted in the Easy-RSA documentation, certificate extensions are selected by the cert type given on the CLI during signing. For this POC, we use the `serverClient` type to allow certificates to be used for both server and client authentication.

The configurations leverage Traefik's ability to handle various certificate formats and sources, demonstrating flexibility in real-world scenarios.

## References

- [Traefik Documentation](https://doc.traefik.io/)
- [Easy-RSA Documentation](https://easy-rsa.readthedocs.io/)
- [OpenSSL Documentation](https://www.openssl.org/docs/)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
