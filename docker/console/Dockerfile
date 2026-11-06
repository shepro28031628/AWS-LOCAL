# Consola web de AWS Nube Local.
#
# Parte de la consola oficial (floci/floci-ui, proyecto aparte cuyo código no está
# en este repo) y solo le cambia la marca: logos, favicon y textos visibles.
#
# Construir:  docker build -t nube-local/floci-ui:0.7.0 docker/console
# El nombre debe terminar en "floci-ui" para que el emulador la reconozca
# como la consola oficial y use su configuración de salud (health check).
FROM floci/floci-ui:0.7.0

COPY nl-logo-light.svg nl-logo-dark.svg nl-icon.svg /app/public/assets/

RUN set -eux; \
    cd /app/public; \
    js=$(ls assets/index-*.js); \
    grep -q 'src:s?Ve:He,alt:`Floci`' "$js"; \
    grep -q 'src:s?Ue:We,' "$js"; \
    sed -i \
      -e 's|src:s?Ve:He,alt:`Floci`|src:s?`/assets/nl-logo-dark.svg`:`/assets/nl-logo-light.svg`,alt:`AWS Nube Local`|' \
      -e 's|src:s?Ue:We,|src:`/assets/nl-icon.svg`,|' \
      -e 's|children:`Local Cloud`|children:`Consola local`|' \
      -e 's|children:`Floci DevTools · Local`|children:`AWS Nube Local · Local`|' \
      "$js"; \
    sed -i \
      -e 's|<title>Floci UI</title>|<title>AWS Nube Local</title>|' \
      -e 's|content="Floci UI[^"]*"|content="AWS Nube Local: consola de tu nube local."|' \
      -e 's|<link rel="icon" href="[^"]*" />|<link rel="icon" href="/assets/nl-icon.svg" type="image/svg+xml" />|' \
      index.html; \
    grep -q 'nl-logo-dark.svg' "$js"; \
    grep -q 'nl-icon.svg' index.html
