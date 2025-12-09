---
layout: post
title: Adding OAuth2 Authentication to Solace Prometheus Exporter
categories: [Development, Open Source, Solace, Container, Prometheus]
---

Contributing OAuth2 authentication support to a community-driven open source project and building comprehensive end-to-end testing infrastructure.

## Background

The [Solace Prometheus Exporter](https://github.com/solacecommunity/solace-prometheus-exporter) is a community-maintained tool that collects metrics from Solace PubSub+ event brokers and exposes them in Prometheus format. While the exporter worked well for basic authentication scenarios, it lacked support for OAuth2 authentication - a critical requirement for modern enterprise environments where security and token-based authentication are standard practices.

During my work with Solace PubSub+ in production environments, I encountered situations where the event brokers were configured to use OAuth2 for authentication. The existing exporter only supported basic username/password authentication, which meant I couldn't monitor these OAuth2-secured instances effectively.

This gap in functionality presented both a problem and an opportunity. Rather than implementing a workaround or switching to a different monitoring solution, I decided to contribute directly to the open source project to benefit the entire community.

## Problem Statement

The challenge was multifaceted:

1. **Missing OAuth2 Support**: The exporter couldn't authenticate against Solace brokers configured with OAuth2 providers like Keycloak, Azure AD, or other identity providers.
2. **Lack of Testing Infrastructure**: The project had limited testing coverage, especially for authentication scenarios.
3. **Configuration Complexity**: OAuth2 introduces additional complexity with token endpoints, client credentials, scopes, and token refresh mechanisms.

The goal was to implement a flexible OAuth2 client that could work with arbitrary OAuth2 providers while maintaining backward compatibility with existing basic authentication setups.

## Implementation Approach

### OAuth2 Client Architecture

I refactored the existing HTTP client logic and extracted authentication concerns into a dedicated `auth.go` module. The implementation supports the OAuth2 client credentials flow, which is the most common pattern for service-to-service authentication.

Key components of the OAuth2 implementation:

```go
type OAuthConfig struct {
    TokenURL     string
    ClientID     string
    ClientSecret string
    Scopes       []string
}
```

The authentication flow works as follows:

1. Authenticate with the OAuth2 provider using client credentials
2. Receive an access token (and optional refresh token)
3. Use the access token in the Authorization header for SEMP API requests
4. Handle token expiry and refresh automatically

### Configuration Enhancement

I extended the configuration structure to support OAuth2 parameters while maintaining backward compatibility:

```ini
[solace]
# Existing basic auth (still supported)
username=admin
password=admin

# New OAuth2 configuration
oauthTokenUrl=https://keycloak.example.com/auth/realms/solace/protocol/openid-connect/token
oauthClientId=solace-prometheus-exporter
oauthClientSecret=secret
oauthScope=SEMP
```

The exporter automatically detects whether to use basic authentication or OAuth2 based on the presence of OAuth2 configuration parameters. While this introduces some automagic, it simplifies user configuration by avoiding the need for separate mode switches.

### HTTP Client Refactoring

The original HTTP client logic was tightly coupled with the main exporter code. I separated these concerns by creating a dedicated `http.go` module that handles:

- TLS configuration
- Authentication (both basic and OAuth2)
- HTTP transport settings
- Request/response handling

This refactoring improved code maintainability and made testing significantly easier.

## Comprehensive Testing Strategy

One of the most challenging aspects of this contribution was building a robust testing infrastructure. I wanted to ensure that the OAuth2 implementation would work reliably in production environments.

### End-to-End Testing Setup

I built what is probably one of the most comprehensive E2E test setups I've ever created for an open source project:

**Docker Compose Test Environment:**

- **Keycloak**: Full identity provider setup with pre-configured realm, clients, and users
- **Solace PubSub+**: Event broker with OAuth2 authentication enabled
- **TLS Infrastructure**: Complete certificate authority with proper certificate chains
- **Test Scripts**: Automated validation of both metric collection and OAuth2 flow

The test environment includes:

```yaml
# docker-compose.yaml (excerpt)
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    volumes:
      - ./prepared-realm.json:/opt/keycloak/data/import/realm.json

  solace:
    image: solace/solace-pubsub-standard:latest
    environment:
      - username_admin_globalaccesslevel=admin
      - username_admin_password=admin
    depends_on:
      - keycloak
```

The keycloak realm was pre-configured by myself manually to include the necessary clients and roles for the tests. I then exported the whole realm configuration to a JSON file and mounted it into the Keycloak container to ensure consistent setup across test runs.

### Test Automation

I implemented automated test scripts that:

1. Wait for all services to be ready
2. Configure Solace with OAuth2 authentication
3. Run the prometheus exporter with OAuth2 configuration
4. Validate that metrics are successfully scraped
5. Verify OAuth2 token exchange and refresh functionality

The CI pipeline now includes this E2E test as a required check, ensuring that future changes won't break OAuth2 functionality.

### Unit Testing

Beyond E2E testing, I added comprehensive unit tests for the configuration parsing logic:

```go
func TestOAuthConfigParsing(t *testing.T) {
    tests := []struct {
        name     string
        config   map[string]string
        expected OAuthConfig
        wantErr  bool
    }{
        // Test cases for various configuration scenarios
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Test implementation
        })
    }
}
```

## Technical Deep Dive

### Token Management

The OAuth2 implementation handles several critical aspects:

**Token Caching**: Access tokens are cached and reused until they expire, reducing unnecessary round trips to the OAuth2 provider.

**Automatic Refresh**: When a token expires, the client automatically requests a new one using the refresh token (if available) or by re-authenticating with client credentials.

**Error Handling**: Robust error handling for various OAuth2 failure scenarios, including network issues, invalid credentials, and token expiry.

### Scope Support

Different Solace deployments may require different OAuth2 scopes. The implementation supports configurable scopes:

```ini
oauthScope=SEMP,READ
```

This flexibility ensures the exporter can work with various OAuth2 provider configurations and permission models.

### TLS and Security

The OAuth2 implementation properly handles TLS configurations, supporting:

- Custom certificate authorities
- Client certificates for mutual TLS
- Certificate verification settings
- Secure token storage and transmission

## Challenges and Learning

### Multi-Service Integration Testing

Building a test environment that reliably brings up Keycloak, Solace, and all the networking components was surprisingly complex. Each service has its own startup timing, dependencies, and configuration requirements.

I learned a lot about:

- Docker networking and service dependencies
- Keycloak realm configuration and export/import
- Solace PubSub+ authentication configuration via SEMP
- Certificate management for test environments

### Go OAuth2 Library Intricacies

Working with Go's `golang.org/x/oauth2` package taught me about proper token management, context handling, and the subtleties of different OAuth2 flows.

### Community Collaboration

This was a significant open source contribution involving:

- Code review processes
- Security scanning (GitGuardian flagged test certificates)
- CI/CD pipeline integration
- Documentation and examples

The project maintainer [@GreenRover](https://github.com/GreenRover) was incredibly supportive throughout the process, providing valuable feedback and helping with repository configuration.

## Results and Impact

The OAuth2 support was successfully merged into the main project ([PR #133](https://github.com/solacecommunity/solace-prometheus-exporter/pull/133)), bringing several benefits:

**For the Community:**

- Enables monitoring of OAuth2-secured Solace deployments
- Provides a comprehensive testing framework for future contributors
- Maintains backward compatibility with existing setups

**For Production Use:**

- Secure authentication without hardcoded passwords
- Token-based authentication aligned with enterprise security practices
- Automatic token refresh reduces operational overhead

**For the Project:**

- Significantly improved test coverage
- Better code organization and maintainability
- Enhanced CI/CD pipeline with automated E2E testing

## Conclusion

Contributing OAuth2 support to the Solace Prometheus Exporter was both challenging and rewarding. It involved not just implementing the OAuth2 flow, but also building comprehensive testing infrastructure and refactoring existing code for better maintainability.

The experience reinforced several important principles:

**Testing is Critical**: Complex integrations require comprehensive testing. The time invested in building the E2E test environment paid dividends in confidence and reliability.

**Community Collaboration**: Open source projects thrive when contributors go beyond just adding features to also improving the overall project quality and testing infrastructure.

**Security First**: Authentication mechanisms need careful consideration of token management, expiry, refresh, and secure storage.

This contribution demonstrates how addressing a specific production need can lead to valuable improvements for the entire community. The OAuth2 support is now available to anyone using the Solace Prometheus Exporter in enterprise environments where token-based authentication is required.

If you're interested in the technical details, you can review the complete implementation in [GitHub PR #133](https://github.com/solacecommunity/solace-prometheus-exporter/pull/133). The testing infrastructure and OAuth2 implementation serve as good examples for other Go projects that need similar authentication capabilities.
