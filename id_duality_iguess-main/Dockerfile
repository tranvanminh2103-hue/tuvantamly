FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN python -m server.cli seed || true
ENV PORT=8000 TRUST_PROXY=1 DB_PATH=/app/data/dualself.db
EXPOSE 8000
CMD ["sh", "-c", "gunicorn -w 2 -b 0.0.0.0:${PORT} gemini_ai_server:app"]
