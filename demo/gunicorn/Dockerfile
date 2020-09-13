FROM ubuntu:18.04

RUN mkdir -p /var/www/TRX/
WORKDIR /var/www/TRX/

# System dependencies
RUN apt-get -yq update &&           \
    apt-get -yq upgrade &&          \
    apt install -y python3-pip

COPY requirements.txt .
RUN pip3 install --upgrade pip &&   \
    pip3 install -r requirements.txt

COPY transforms/ transforms/
COPY project.py .

RUN chown -R www-data:www-data /var/www/TRX/

EXPOSE 8080
ENTRYPOINT ["gunicorn", "--bind=0.0.0.0:8080", "--threads=25", "--workers=2", "project:app"]