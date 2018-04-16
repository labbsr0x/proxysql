# ProxySQL
Docker image for ProxySQL built from Ubuntu Packages

## How to run this container
Simple: `docker run -d -p 6032:6032 -p 6033:6033 labbsr0x/proxysql:1.4.8`

But for a more complete comprehensive example, head to (https://github.com/labbsr0x/proxysql/tree/master/examples/simple) and check a complete example with 1 ProxySQL and 3 MySQL Backends with 1 read-only instance and regex Query Rule being applied
## How to build this image

Choose the folder of the ProxySQL version you want to build the image and execute: `docker-compose build`

