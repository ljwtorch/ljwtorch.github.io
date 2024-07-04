[官方文档](https://docs.drone.io/)](https://docs.drone.io/)

drone-server+ssh-runner 

docker compose 部署

```yaml
services:
  drone:
    image: drone/drone:latest
    volumes:
      - ./:/data
    environment:
      - DRONE_GITHUB_CLIENT_ID=
      - DRONE_GITHUB_CLIENT_SECRET=
      - DRONE_RPC_SECRET=
      - DRONE_SERVER_HOST=
      - DRONE_SERVER_PROTO=http
    ports:
      - "8085:80"
      - "4433:443"
    restart: always
    container_name: drone

  drone-runner:
    image: drone/drone-runner-ssh
    environment:
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_HOST=
      - DRONE_RPC_SECRET=
    ports:
      - "3001:3000"
    restart: always
    container_name: runner-ssh
```
![Uploading 31BEE20F-9ED6-471A-BF70-7C488FF1BF71.png…]()
