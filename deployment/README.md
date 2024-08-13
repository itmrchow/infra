# local執行drone

## 1. 執行Ngrok
```bash
ngrok http http://localhost:8081
```

## 2. 複製Ngrok的url

## 3. 修改gitlab回調url

## 4. 執行Docker-compose
```bash
docker-compose -f deployment/docker-compose.yml up
```

## 5. 執行修改drone各個服務setting
```bash
deployment/local/.drone_local.yml
```


測試1 測試2 測試3 測試4