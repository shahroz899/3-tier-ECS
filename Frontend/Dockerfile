FROM node:16 as builder
WORKDIR /app
COPY .  .
RUN npm install
RUN npm run build


FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/build /usr/share/nginx/html

ARG REACT_APP_PUBLIC_URL 
ENV REACT_APP_PUBLIC_URL=REACT_APP_PUBLIC_URL 

EXPOSE 3000
CMD ["nginx","-g","daemon off;"]
