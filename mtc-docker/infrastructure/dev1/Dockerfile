# Original Command: 
# docker run -dp 5000:5000 -v $PWD/app:/app -w /app python /bin/sh run.sh

FROM python
 
COPY ./app/ /app
 
WORKDIR /app
 
RUN pip install -r requirements.txt
 
EXPOSE 5000
 
ENTRYPOINT [ "python", "app.py" ]