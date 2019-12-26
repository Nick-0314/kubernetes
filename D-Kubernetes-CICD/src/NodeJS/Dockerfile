FROM harbor.devops.com/devops/nginx:tt
COPY ./build /usr/local/nginx/html
ENTRYPOINT ["nginx"]
CMD ["-g","daemon off;"]
