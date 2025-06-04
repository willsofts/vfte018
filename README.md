# Tenant Setting

This project introduce to use Vue 3 framework create single page application.

## How To Install

This project is under [@willsofts](https://github.com/willsofts) libraries protection as private access, then you have to gain access key from administrator and setting in your own environment before start installation. \
ex. \
Window

    set NPM_TOKEN=your access token key here

Linux

    export NPM_TOKEN=your access token key here

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).

### How To Examine
1. start [vueserve](https://github.com/willsofts/vueserve) project 
    goto https://github.com/willsofts/vueserve
2. api try out, this is supported api that you have created ex. \
    curl -X POST http://localhost:8080/api/sfte018/list

This vue application using `sfte018` api to manipulate transaction. \
In development mode you can setting environment before start up project

    set VALIDATE_TOKEN=false

or

    export VALIDATE_TOKEN=false

### Testing
1. Normally copy output folder after build success ex. vfte018 into [vueserve](https://github.com/willsofts/vueserve) project under public path and then you can launch it directly ex. 

    http://localhost:8080/vfte018/vfte018.html

2. This can run serve or using file protocol testing but it must build with environment variables,
try to set below before startup build command.

```
    set VUE_APP_API_URL=http://127.0.0.1:8080
    set VUE_APP_BASE_URL=http://127.0.0.1:8080
```

In case of validate token you need to specified VUE_APP_API_TOKEN too.

3. Using `app.config.js` by apply setting appVariables like
```
    API_URL: "http://127.0.0.1:8080",
    BASE_URL: "http://127.0.0.1:8080",
```
    
In case of validate token you need to specified API_TOKEN too.
