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

## Proyectos enrutados

| Proyecto | Subdominio | Contenedor | Repo |
|---|---|---|---|
| Portafolio | prittor.com / www | portafolio-app | Caro2870/portafolio-2026 |
| Finanzas | finanzas/caro/sam/jessica/demo.prittor.com | finanzas-backend-prod, finanzas-frontend-prod | Caro2870/finanzas |
| Atiende | atiende.prittor.com | atiende-atiende-1 | Caro2870/atiende |
| Jessica Nails | shop/tienda.prittor.com | jessica-nails-app | Caro2870/jessica-nails |
| Ecommerce Nails | nails.prittor.com | ecommerce-nails-app | Caro2870/ecommerce-nails |
| LLM Visualizer | llm.prittor.com | llm-visualizer-app-1 | Caro2870/llm-visualizer |
| Voice Notes | voice.prittor.com | voice-notes-app | Caro2870/voice-notes |
