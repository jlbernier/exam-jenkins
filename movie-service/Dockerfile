FROM python:3.8-slim

WORKDIR /app

COPY ./requirements.txt /app/requirements.txt

RUN apt-get update \
    && apt-get install -y gcc \
    && apt-get clean

RUN pip install -r /app/requirements.txt \
    && rm -rf /root/.cache/pip

COPY . /app/

# (facultatif mais utile)
EXPOSE 8000

# Lancement du serveur FastAPI
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
