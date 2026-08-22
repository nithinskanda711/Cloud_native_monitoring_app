from python:3.12-slim-buster

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

copy . .

ENV FLASK_RUN_HOST=0.0.0.0
 
EXPOSE 5001

CMD [ "flash", "run" ]
