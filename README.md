# ConfigData — Public Configuration API and Usage Guide

This repository is a centralized configuration source for services that consume configuration via Spring Cloud Config Server. It exposes a public configuration surface in the form of well-defined keys organized by application name and profile.


## Repository structure
- `application.yml`: Settings shared by all applications and all profiles.
- `lab-3-client.yml`: Default configuration for the `lab-3-client` application.
- `lab-3-client-northamerica.yml`: Profile-specific overrides for `lab-3-client` when the `northamerica` profile is active.


## Naming conventions
- File names follow `{application}.yml` and `{application}-{profile}.yml`.
- The special `application.yml` applies to every application and every profile.
- Git branch acts as the Config Server "label". If you serve from `main`, the label is `main` (defaults vary by server config).


## Public configuration keys

### Global (from `application.yml`)
- `eureka.client.serviceUrl.defaultZone` (string)
  - Description: Discovery server URL for Eureka clients.
  - Default in this repo: `http://localhost:8010/eureka/`

### lab-3-client
- `luckly-word` (string)
  - Description: A sample phrase used by the `lab-3-client` service. Note the key is intentionally spelled `luckly-word` to match clients.
  - Default (from `lab-3-client.yml`): `Rabbit's Foot`
  - Override (from `lab-3-client-northamerica.yml`, profile `northamerica`): `Irish`


## How clients consume this configuration

### 1) Via Spring Cloud Config Server (recommended)
1. Run a Config Server pointing to this repository.
2. In each client application:
   - Set the application name to match a file prefix in this repo, for example:
     - `spring.application.name=lab-3-client`
   - Import the Config Server:
     - Spring Boot 2.4+ (typical): `spring.config.import=optional:configserver:http://localhost:8888`
     - Older setups: use `bootstrap.yml`/`bootstrap.properties` with `spring.cloud.config.uri`.
   - Activate profiles as needed (e.g., `northamerica`) via `spring.profiles.active=northamerica` or environment variables.

When the client starts, properties are resolved in this order:
1. `application.yml` (global defaults in this repo)
2. `{application}.yml` (e.g., `lab-3-client.yml`)
3. `{application}-{profile}.yml` (e.g., `lab-3-client-northamerica.yml`)
4. Local overrides (env vars, command line, local files), per Spring precedence rules

Code example (Spring Boot):
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class LuckyWordController {
    @Value("${luckly-word:World}")
    private String luckyWord;

    @GetMapping("/lucky")
    public String lucky() {
        return luckyWord;
    }
}
```

### 2) Fetching raw config over HTTP (from the Config Server)
The Config Server exposes endpoints of the form:
- `/{application}/{profile}`
- `/{application}/{profile}/{label}`

Examples:
- `GET http://localhost:8888/lab-3-client/default` → default config for `lab-3-client`
- `GET http://localhost:8888/lab-3-client/northamerica` → includes the `northamerica` overrides
- `GET http://localhost:8888/application/default` → global settings (e.g., `eureka.client.serviceUrl.defaultZone`)


## Usage examples

### Example: retrieving `luckly-word` in a service
- Default profile:
  - Will resolve to `Rabbit's Foot` from `lab-3-client.yml`.
- With `spring.profiles.active=northamerica`:
  - Will resolve to `Irish` from `lab-3-client-northamerica.yml`.

### Example: using Eureka default zone
Clients with Eureka enabled will pick up:
- `eureka.client.serviceUrl.defaultZone = http://localhost:8010/eureka/`
You can override this per environment using profile-specific files, e.g., create `application-prod.yml` with a different URL.


## Adding or changing configuration
- Add a new app: create `{app}.yml` and optional `{app}-{profile}.yml` files.
- Add a new profile override: create `{app}-{profile}.yml` with only the keys you need to override.
- Do not duplicate keys unnecessarily; rely on cascading from `application.yml` and `{app}.yml`.
- Prefer kebab-case for keys to match common Spring conventions already in use here (e.g., `luckly-word`).


## Best practices
- Do not commit secrets. Use a secrets manager or Config Server encryption support.
- Document new keys with clear descriptions and expected types.
- Keep defaults sensible; prefer environment-specific overrides for variations.
- Use labels (branches/tags) to version configuration safely across deployments.


## Troubleshooting
- Client does not see updates: ensure `spring.cloud.config.allowOverride=true` and refresh scope if required (`/actuator/refresh` with Spring Cloud), or restart the service.
- Wrong application name: verify `spring.application.name` matches the `{application}` portion of files in this repo.
- Wrong profile: verify `spring.profiles.active` and that the `{application}-{profile}.yml` exists.


## Glossary
- Application: the logical service name (e.g., `lab-3-client`).
- Profile: an environment or region-specific variant (e.g., `northamerica`, `prod`).
- Label: the Git branch or tag served by the Config Server (e.g., `main`).
