FROM python:alpine

WORKDIR /app/netflix

COPY . .

RUN pip install --no-cache-dir -r requirements.txt \
&& pip install django django-allauth\
&& python -m pip install Pillow  \
&& python manage.py migrate 

EXPOSE 8000

CMD [ "python", "manage.py", "runserver", "0.0.0.0:8000" ]