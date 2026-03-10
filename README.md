# prittor-infra

Infraestructura centralizada de Prittor. Contiene el reverse proxy (Caddy) que enruta todos los subdominios de `prittor.com`.

## Estructura

```
caddy/
└── Caddyfile       ← todos los subdominios aquí
docker-compose.yml  ← solo Caddy + red compartida
```

## Red compartida

Todos los proyectos deben conectarse a la red `prittor-proxy` para que Caddy pueda enrutarlos.

## Agregar un nuevo subdominio

1. Editar `caddy/Caddyfile` y agregar el bloque
2. Conectar el contenedor del proyecto a la red `prittor-proxy`
3. Recargar Caddy: `docker exec prittor-caddy caddy reload --config /etc/caddy/Caddyfile`

## Deploy inicial

```bash
git clone git@github.com:Caro2870/prittor-infra.git /opt/prittor-infra
cd /opt/prittor-infra
docker compose up -d
```

## Conectar contenedores existentes a la red

```bash
docker network connect prittor-proxy <nombre-contenedor>
```
