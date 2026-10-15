# ---------- Estágio 1: build (instala dependências) ----------
FROM python:3.11-slim AS builder

WORKDIR /build
COPY requirements.txt .
# --prefix=/install deixa as libs isoladas para copiarmos só elas para a imagem final
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---------- Estágio 2: imagem final (menor e sem ferramentas de build) ----------
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1 \
    PORT=8005

# Usuário sem privilégios (boa prática de segurança)
RUN useradd --create-home --uid 1000 appuser
WORKDIR /app

COPY --from=builder /install /usr/local
COPY app.py .

USER appuser
EXPOSE 8005

# IMPORTANTE: 1 worker só. O app inicia uma thread que consome a SQS ao importar;
# com N workers do gunicorn teríamos N consumidores dentro do mesmo container.
# Quem escala é o HPA (mais pods), não o gunicorn.
CMD ["gunicorn", "--workers", "1", "--bind", "0.0.0.0:8005", "app:app"]
