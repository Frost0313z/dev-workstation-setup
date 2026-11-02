FROM nginx:alpine
LABEL org.opencontainers.image.title="my-web"
LABEL org.opencontainers.image.authors="Frost0313z"
ENV APP_ENV=dev
COPY site/ /usr/share/nginx/html/
EXPOSE 80
HEALTHCHECK --interval=10s --timeout=3s CMD wget -qO- http://localhost/ || exit 1