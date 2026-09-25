# Documentação de Integração: API Estática de Arquivos JSON para Zabbix

Esta documentação descreve a arquitetura e os passos de configuração no RHEL para expor arquivos JSON estáticos de dispositivos para consumo no Zabbix. A solução utiliza Gunicorn/Uvicorn (Python) isolado em ambiente virtual, NGINX como proxy reverso modular com bloqueio de IP e Zabbix HTTP Agent.

## Boas Práticas Adotadas (Data vs Code)
Para facilitar versionamento, backups e segurança, a arquitetura física foi separada em duas frentes:
* **Código (`/zabbix_api/app/api/`):** Contém os scripts, ambiente virtual e dependências.
* **Dados (`/zabbix_api/data/devices/`):** Contém exclusivamente os arquivos JSON gerados pelo seu sistema.

```text
/zabbix_api/
├── app/
│   └── api/                <-- Apenas Código (main.py, venv)
└── data/
    └── devices/            <-- Apenas Dados (Pastas unity, isilon, etc)
```

---

## 1. Preparação do Ambiente RHEL e Refatoração

Para não interferir no Python global do sistema e garantir máxima segurança, criamos um usuário de serviço sem permissão de login e um ambiente virtual (venv).

**1.1. Criar usuário de serviço (caso ainda não exista):**
```bash
sudo useradd -r -s /bin/false zabbix_api
```

**1.2. Criar a nova estrutura de pastas separada:**
```bash
sudo mkdir -p /zabbix_api/app/api
sudo mkdir -p /zabbix_api/data/devices

# Se você já tem a pasta devices antiga dentro de api, mova-a:
# sudo mv /zabbix_api/app/api/devices/* /zabbix_api/data/devices/

# Ajuste as permissões de tudo para o usuário de serviço
sudo chown -R zabbix_api:zabbix_api /zabbix_api
```

**1.3. Criar Ambiente Virtual e dependências:**
```bash
cd /zabbix_api/app/api
sudo -u zabbix_api python3 -m venv venv
sudo -u zabbix_api venv/bin/pip install fastapi uvicorn gunicorn

# Criar o arquivo de requirements para registrar as versões exatas
sudo -u zabbix_api venv/bin/pip freeze > requirements.txt
```

---

## 2. Código da API (FastAPI)

Crie ou atualize o arquivo `/zabbix_api/app/api/main.py`. Note que o `BASE_PATH` agora aponta para a nova pasta de dados.

```python
import os
import json
from fastapi import FastAPI, HTTPException, Header

app = FastAPI()

# Caminho absoluto ATUALIZADO para a nova pasta de dados
BASE_PATH = "/zabbix_api/data/devices"

# Token de segurança esperado do Zabbix
SECRET_TOKEN = "seu_token_super_secreto_123"

@app.get("/devices/{device_family}/{device_name}")
def get_device_json(
    device_family: str, 
    device_name: str, 
    x_api_token: str = Header(None)
):
    """
    Lê o arquivo JSON específico do dispositivo e retorna seu conteúdo.
    Bloqueia a requisição caso o token não seja fornecido ou esteja incorreto.
    """
    if x_api_token != SECRET_TOKEN:
        raise HTTPException(status_code=401, detail="Não autorizado: Token inválido ou ausente")

    file_path = f"{BASE_PATH}/{device_family}/{device_name}/public/done-{device_name}.json"
    
    if not os.path.exists(file_path):
        raise HTTPException(status_code=404, detail="JSON do dispositivo não encontrado")
    
    with open(file_path, "r", encoding="utf-8") as json_file:
        device_data = json.load(json_file)
        
    return device_data
```

---

## 3. Configuração do Serviço Systemd (Gunicorn)

**3.1. Criar ou editar arquivo de serviço:**
```bash
sudo vim /etc/systemd/system/zabbix-api.service
```

**3.2. Configuração (O código continua na pasta api):**
```ini
[Unit]
Description=API Zabbix Leitora de JSON
After=network.target

[Service]
User=zabbix_api
Group=zabbix_api
WorkingDirectory=/zabbix_api/app/api

# Executa o Gunicorn direto de dentro do Ambiente Virtual (venv)
ExecStart=/zabbix_api/app/api/venv/bin/gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 127.0.0.1:8000

Restart=always

[Install]
WantedBy=multi-user.target
```

**3.3. Iniciar e aplicar mudanças no serviço:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart zabbix-api
sudo systemctl enable zabbix-api
```

---

## 4. Segurança e Roteamento (NGINX Modular no RHEL)

**4.1. Criar rota exclusiva da API:**
```bash
sudo vim /etc/nginx/default.d/zabbix_api.conf
```

**4.2. Configuração (Proteção por IP):**
```nginx
location /devices/ {
    # Lista de IPs permitidos
    allow 127.0.0.1;           # Testes locais no servidor
    allow 192.168.1.50;        # Exemplo: IP do Server/Proxy Zabbix
    
    # IMPORTANTE: Descomente a linha abaixo em producao para bloquear acessos externos não listados acima
    # deny all;

    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

**4.3. Liberar o Proxy no SELinux (CRÍTICO EM RHEL):**
```bash
sudo setsebool -P httpd_can_network_connect 1
```

**4.4. Validar e Recarregar NGINX:**
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 5. Como Testar a API

Antes de ir para o Zabbix, crie um arquivo na **nova pasta de dados** e simule a chamada.

### Criando o dado falso para teste:
```bash
sudo -u zabbix_api mkdir -p /zabbix_api/data/devices/unity/unityrio001/public/
sudo -u zabbix_api bash -c 'echo "{\"status\": \"ok\", \"cpu_utilization\": 45}" > /zabbix_api/data/devices/unity/unityrio001/public/done-unityrio001.json'
```

### Teste via Terminal (cURL):
```bash
curl -X GET "http://127.0.0.1/devices/unity/unityrio001" \
     -H "x-api-token: seu_token_super_secreto_123" | jq .
```

---

## 6. Coleta no Zabbix

### 6.1. Item Master (O Coletor Principal)
* **Type:** `HTTP agent`
* **Key:** `api.get_json_data`
* **URL:** `http://ip_do_seu_nginx/devices/unity/{HOST.NAME}`
* **Headers:** Adicionar Name `x-api-token` e Value `{$API_TOKEN}` (Macro).
* **History storage period:** `0` (Não armazena os textos brutos no banco).

### 6.2. Itens Dependentes (Extração das Métricas)
* **Type:** `Dependent item`
* **Master item:** `API: Obter dados do equipamento`
* **Preprocessing:** 
  * Step: `JSONPath`
  * Parameters: `$.cpu_utilization`
  * Ativar **Custom on fail** -> `Discard value`.%   
