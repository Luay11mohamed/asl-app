FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

# Shell form (not exec/JSON form) is required so $PORT actually gets
# expanded — Railway assigns a dynamic PORT per deployment and routes
# external traffic to it. The exec-form ["uvicorn", ..., "--port", "8000"]
# never expands env vars, so it always bound to 8000 regardless of what
# Railway actually wanted, which can silently break routing on redeploy.
# ${PORT:-8000} keeps a sane default for local `docker run` / compose.
CMD uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}