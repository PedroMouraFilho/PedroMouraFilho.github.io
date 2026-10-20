# syntax=docker/dockerfile:1

# =========================================================================
# Dockerfile — Site Nuxt (app-pedro) servido como estático via Nginx
# -------------------------------------------------------------------------
# O projeto é gerado como site estático (pnpm generate -> .output/public)
# e servido por um Nginx enxuto. Multi-stage para a imagem final ser leve.
# =========================================================================

# ---------- Estágio 1: build do site estático ----------
FROM node:24-alpine AS builder

# pnpm via corepack (versão fixada no package.json -> packageManager)
RUN corepack enable

WORKDIR /app

# Instala dependências primeiro para aproveitar o cache de camadas
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile

# Copia o restante do código e gera o site estático
COPY . .
RUN pnpm generate

# Salvaguarda contra o bug do Nuxt 4 / nitropack 2.13 (estrutura app/) em que os
# publicAssets — o client build (a pasta _nuxt/ com CSS/JS) e a pasta public/ —
# às vezes NÃO são copiados para .output/public, gerando página sem estilo (404).
#
# O diretório do client build muda de lugar conforme o ambiente: pode estar em
# .nuxt/dist/client/_nuxt (como no dev local) ou em
# node_modules/.cache/nuxt/.nuxt/dist/client/_nuxt (como no build Linux/CI).
# Em vez de fixar um caminho, localizamos dinamicamente a pasta _nuxt que contém
# os assets client e a copiamos para .output/public/_nuxt. Também copiamos
# public/. (favicon, imagens). O build falha cedo se o _nuxt não for encontrado,
# evitando publicar uma versão quebrada.
RUN set -eux; \
    mkdir -p .output/public; \
    # Arquivos públicos estáticos (favicon, imagens em img/, etc.)
    if [ -d public ]; then \
      cp -R public/. .output/public/; \
    fi; \
    # Se o generate já emitiu o _nuxt, nada a fazer. Senão, procuramos a pasta
    # _nuxt do client build (a que contém o entry*.css) e a copiamos.
    if [ ! -d .output/public/_nuxt ]; then \
      src="$(find .nuxt node_modules/.cache -type d -path '*/dist/client/_nuxt' 2>/dev/null | head -n 1)"; \
      if [ -z "$src" ]; then \
        src="$(dirname "$(find .nuxt node_modules/.cache -type f -path '*/_nuxt/entry*.css' 2>/dev/null | head -n 1)")"; \
      fi; \
      echo "Fonte do _nuxt encontrada: '$src'"; \
      test -n "$src"; \
      mkdir -p .output/public/_nuxt; \
      cp -R "$src"/. .output/public/_nuxt/; \
    fi; \
    # Garante que o CSS de entrada exista — senão aborta o build
    ls .output/public/_nuxt/entry*.css; \
    echo "OK: _nuxt presente em .output/public"

# ---------- Estágio 2: imagem final com Nginx ----------
FROM nginx:1.27-alpine AS runner

# Config do Nginx escutando na porta 82
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Artefato estático gerado no estágio anterior
COPY --from=builder /app/.output/public /usr/share/nginx/html

EXPOSE 82

# Healthcheck simples: verifica se o Nginx responde na porta 82
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://127.0.0.1:82/ >/dev/null 2>&1 || exit 1

CMD ["nginx", "-g", "daemon off;"]
