FROM registry.access.redhat.com/ubi10/nginx-126:10.0

# Ta bort default content
RUN rm -rf /opt/app-root/src/*

# Kopiera HTML
COPY index.html /opt/app-root/src/index.html

# Kopiera nginx config SIST och till rätt plats
COPY nginx.conf /etc/nginx/nginx.conf

# Sätt rätt permissions
USER root
RUN chmod 644 /etc/nginx/nginx.conf
USER 1001

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
