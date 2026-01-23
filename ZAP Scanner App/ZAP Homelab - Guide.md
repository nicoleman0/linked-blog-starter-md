## Pull the image

`docker pull zaproxy/zap-stable`

## Start in headless daemon mode

`docker run -d --name zap-daemon --network host zaproxy/zap-stable zap.sh -daemon -host 0.0.0.0 -port 8080 -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true -config api.key=changeme`

# Commands

## Scanning Workflow

### 1. Spider scan (discovery) the Juice Shop
```bash
curl "http://localhost:8080/JSON/spider/action/scan/?apikey=changeme&url=http://localhost:3000"
```

### 2. Check spider progress
```bash
curl "http://localhost:8080/JSON/spider/view/status/?apikey=changeme"
```

### 3. Run active scan (after spidering)
```bash
curl "http://localhost:8080/JSON/ascan/action/scan/?apikey=changeme&url=http://localhost:3000&recurse=true"
```

### 4. Check active scan progress
```bash
curl "http://localhost:8080/JSON/ascan/view/status/?apikey=changeme"
```

### 5. Get alerts/vulnerabilities
```bash
curl "http://localhost:8080/JSON/core/view/alerts/?apikey=changeme&baseurl=http://localhost:3000"
```

### 6. Generate HTML report
```bash
curl "http://localhost:8080/OTHER/core/other/htmlreport/?apikey=changeme" -o zap-report.html
```

## Managing the Daemon

### Stop
```bash
docker stop zap-daemon
```

### Start
```bash
docker start zap-daemon
```

### Remove
```bash
docker rm -f zap-daemon
```

### View logs
```bash
docker logs zap-daemon
```