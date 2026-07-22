FROM python:3.10-slim

WORKDIR /app

# Install system dependencies required to build psycopg2 and connect to PostgreSQL.
RUN apt-get update -y && \
    apt-get install -y build-essential libpq-dev gcc && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Copy dependency list first to leverage Docker layer caching.
COPY requirements.txt .

RUN pip install --upgrade pip setuptools wheel && \
    pip install --no-cache-dir -r requirements.txt

COPY . .

# The Flask app listens on port 5153 (defined in app.py).
EXPOSE 5153

CMD ["python", "app.py"]