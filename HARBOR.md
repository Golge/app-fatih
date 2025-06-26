# Harbor Registry Configuration

Harbor registry is now properly configured for public access:

- **Internal Access**: http://10.240.0.2:30083
- **External Access**: http://34.32.141.92:30083
- **Admin Credentials**: admin / Harbor12345

## GitHub Actions Access

GitHub Actions can now connect to Harbor using the external IP address.

## Local Access

For local development, configure Docker for insecure registry:

```bash
echo '{"insecure-registries":["34.32.141.92:30083","10.240.0.2:30083"]}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
```

## Docker Login

```bash
docker login 34.32.141.92:30083 -u admin -p Harbor12345
```
