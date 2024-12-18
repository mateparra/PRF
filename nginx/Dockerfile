FROM nginx:latest

#Copiar el archivo de configuración de NGINX
COPY nginx.conf /etc/nginx/nginx.conf

#Copiar la página principal personalizada
COPY site.html /usr/share/nginx/html/site.html

#Exponer el puerto 80
EXPOSE 80