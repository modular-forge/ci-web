# ci-web

Workflow reutilizable de GitHub Actions para las webs estáticas de `modular-forge`. Verifica y compila la web una sola vez y, con cada push a `main`, la despliega en el servidor doméstico (Raspberry Pi) y comprueba desde Internet que la versión nueva está publicada.

## Uso

En el proyecto, `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions: {}

jobs:
  web:
    uses: modular-forge/ci-web/.github/workflows/web.yml@<sha-completo> # vX.Y.Z
    permissions:
      contents: read
      id-token: write
    secrets:
      DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
    with:
      site: <sitio>
      dist: dist/<proyecto>/browser
      url: https://<dominio>
      ts-oauth-client-id: ${{ vars.TS_OAUTH_CLIENT_ID }}
      ts-audience: ${{ vars.TS_AUDIENCE }}
      known-hosts: ${{ vars.FORGE_KNOWN_HOSTS }}
```

Se fija siempre por SHA completo, con la etiqueta en un comentario, para que Dependabot (ecosistema `github-actions`) proponga las versiones nuevas.

## Qué necesita el proyecto

- **`.nvmrc`** con la versión de Node.
- **Entorno `production`**, limitado a la rama `main`, con el secreto `DEPLOY_SSH_KEY`: la clave privada SSH del usuario `deploy` de la Pi.
- **Variables del repositorio:** `TS_OAUTH_CLIENT_ID` y `TS_AUDIENCE` (credencial federada de Tailscale con `tag:ci`, cuyo *subject* es `repo:<propietario>/<repo>:environment:production`) y `FORGE_KNOWN_HOSTS` (línea de `known_hosts` con la clave de host de la Pi).
- **En la Pi:** `/srv/www/<sitio>/releases` y el sitio en la lista `SITIOS` de `forge-desplegar`.

## Contrato de scripts de npm

El job `verify` ejecuta estos scripts si existen (`--if-present`), en este orden: `lint`, `lint:styles`, `format:check`, `deps:validate`, `knip`, `type-coverage`, `test:ci`, `i18n:check`. Después, `build` (obligatorio).

## Qué hace

1. **`verify`** (en PR y en push): `npm ci`, scripts del contrato, compilación y comprobación de la salida (`index.html` presente, sin enlaces simbólicos). Añade `version.json` con el commit y guarda la web como artefacto.
2. **`deploy`** (solo con push a `main`, en el entorno `production`):
   - Entra en la tailnet como nodo efímero con `tag:ci`, sin secretos de Tailscale.
   - Sube la web por `rsync` a `entrante/<sitio>.<commit>/` y después la petición a `peticiones/`. La clave solo puede escribir en `/srv/deploy`: ni leer ni borrar.
   - La Pi valida, copia y activa la versión, y avisa por ntfy.
   - Espera a que `https://<dominio>/version.json` tenga el commit nuevo y comprueba las cabeceras de seguridad.

## Versiones

Etiquetas `vX.Y.Z`. Un cambio que obligue a tocar el `ci.yml` de los proyectos (entradas nuevas obligatorias, nombres distintos) sube la versión mayor.
