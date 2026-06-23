FROM node:24-alpine AS builder

RUN npm install -g openclaw@latest

FROM node:24-alpine

RUN apk add --no-cache \
    tini \
    git \
    curl

COPY --from=builder /usr/local/lib/node_modules /usr/local/lib/node_modules
COPY --from=builder /usr/local/bin /usr/local/bin

RUN mkdir -p /etc/openclaw /workspace/memory /workspace/skills /workspace/logs && \
    chown -R node:node /etc/openclaw /workspace

USER node

ENV OPENCLAW_CONFIG_PATH=/etc/openclaw/openclaw.json
ENV OPENCLAW_WORKSPACE_PATH=/workspace
ENV NODE_ENV=production

EXPOSE 3000 3001

ENTRYPOINT ["tini", "--"]
CMD ["openclaw", "gateway", "--port", "3000"]
