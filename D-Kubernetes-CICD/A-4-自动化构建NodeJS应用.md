# 自动构建NodeJS应用

本节介绍自动化构建NodeJS应用，其构建方式和自动化java基本相同，重点是更改Deployment，Jenkinsfile和Dockerfile

## 1 定义Dockerfile

如果NodeJS仅仅作为前端，在使用NPM进行编译后，一般在dist目录会生成相应的html文件，可以直接使用Nginx进行部署

```
FROM harbor.devops.com/devops/nginx:tt
COPY ./dist /usr/local/nginx/html
ENTRYPOINT ["nginx"]
CMD ["-g","daemon off;"]
```

