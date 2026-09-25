# Documentação de Integração: API Estática de Arquivos JSON para Zabbix

A documentação definitiva contempla todo o ciclo de vida da aplicação: desde a preparação no ambiente com internet até o empacotamento, transferência e instalação *offline* no servidor de produção final.

Esta documentação descreve a arquitetura e os passos de configuração no RHEL para expor arquivos JSON estáticos de dispositivos para consumo no Zabbix. A solução foca em ambientes corporativos isolados (sem internet), utilizando deploy offline via pacotes `.whl`, Gunicorn/Uvicorn em ambiente virtual (venv), NGINX como proxy reverso com bloqueio de IP e Zabbix HTTP Agent.

## Boas Práticas Adotadas (Data vs Code)
Para facilitar versionamento, backups e segurança, a arquitetura física foi separada em duas frentes:
* **Código (`/zabbix_api/app/api/`):** Contém os scripts, ambiente virtual e dependências.
* **Dados (`/zabbix_api/data/devices/`):** Contém exclusivamente os arquivos JSON gerados pelo seu sistema.

---

## 1. Arquitetura de Diretórios

O código-fonte é estritamente separado dos dados gerados (JSON) e das dependências offline.

```text
/zabbix_api/
├── app/
│   └── api/                <-- Código-Fonte e Motor
│       ├── main.py
│       ├── requirements.txt
│       ├── offline_packages/ <-- Pacotes Python (.whl) para instalacao sem internet
│       └── venv/             <-- Ambiente Virtual (Nao deve ser copiado entre servidores)
└── data/
    └── devices/            <-- Dados (Arquivos JSON expostos para o Zabbix)
        ├── unity/
        └── isilon/

```

---

## 2. Preparação e Empacotamento (Servidor de Origem - Com Internet)

Estes passos devem ser executados no ambiente de testes/desenvolvimento para preparar o pacote que será enviado para produção.

**2.1. Preparar o ambiente base:**

```bash
sudo useradd -r -s /bin/false zabbix_api
sudo mkdir -p /zabbix_api/app/api
sudo mkdir -p /zabbix_api/data/devices
sudo chown -R zabbix_api:zabbix_api /zabbix_api

```

**2.2. Instalar dependências e gerar lista (requirements):**

```bash
cd /zabbix_api/app/api
sudo -u zabbix_api python3 -m venv venv
sudo -u zabbix_api venv/bin/pip install fastapi uvicorn gunicorn
sudo -u zabbix_api venv/bin/pip freeze > requirements.txt

```

**2.3. Baixar pacotes para instalação Offline:**

```bash
sudo -u zabbix_api mkdir offline_packages
sudo -u zabbix_api venv/bin/pip download --no-cache-dir -r requirements.txt -d offline_packages/

```

**2.4. Gerar o pacote de Deploy (Ignorando o venv):**
O ambiente virtual não é portátil. Ele deve ser excluído do pacote e recriado no destino.

```bash
cd /
sudo tar -czvf zabbix_api_deploy_offline.tar.gz \
    --exclude='zabbix_api/app/api/venv' \
    --exclude='zabbix_api/app/api/__pycache__' \
    zabbix_api/

```

---

## 3. Deploy no Servidor de Produção (Servidor de Destino - Sem Internet)

Transfira o arquivo `zabbix_api_deploy_offline.tar.gz` para o servidor RHEL de produção e execute os passos abaixo.

**3.1. Extrair arquivos e criar usuário de serviço:**

```bash
sudo tar -xzvf zabbix_api_deploy_offline.tar.gz -C /
sudo useradd -r -s /bin/false zabbix_api
sudo chown -R zabbix_api:zabbix_api /zabbix_api

```

**3.2. Recriar o Ambiente Virtual e Instalar Offline:**

```bash
cd /zabbix_api/app/api
sudo -u zabbix_api python3 -m venv venv

# Instala lendo os pacotes locais, sem tentar acessar a internet (--no-index)
sudo -u zabbix_api venv/bin/pip install --no-index --find-links=offline_packages/ -r requirements.txt

```

---

## 4. Código da API (FastAPI)

Arquivo: `/zabbix_api/app/api/main.py`

```python
import os
import json
from fastapi import FastAPI, HTTPException, Header

app = FastAPI()

# Caminho absoluto para a raiz dos arquivos de dados
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
    Le o arquivo JSON especifico do dispositivo e retorna seu conteudo.
    Bloqueia a requisicao caso o token nao seja fornecido ou esteja incorreto.
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

## 5. Configuração do Serviço Systemd (Gunicorn)

**5.1. Criar arquivo de serviço:**

```bash
sudo vim /etc/systemd/system/zabbix-api.service

```

**5.2. Configuração:**

```ini
[Unit]
Description=API Zabbix Leitora de JSON
After=network.target

[Service]
User=zabbix_api
Group=zabbix_api
WorkingDirectory=/zabbix_api/app/api

ExecStart=/zabbix_api/app/api/venv/bin/gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 127.0.0.1:8000
Restart=always

[Install]
WantedBy=multi-user.target

```

**5.3. Iniciar serviço:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable zabbix-api
sudo systemctl start zabbix-api

```

---

## 6. Segurança e Roteamento (NGINX Modular no RHEL)

**6.1. Criar rota exclusiva da API:**

```bash
sudo vim /etc/nginx/default.d/zabbix_api.conf

```

**6.2. Configuração (Proteção por IP):**

```nginx
location /devices/ {
    allow 127.0.0.1;           # Testes locais no servidor
    allow 192.168.1.50;        # Exemplo: IP do Server/Proxy Zabbix
    
    # ATENCAO: Descomente em producao para bloquear acessos nao autorizados
    # deny all;

    proxy_pass [http://127.0.0.1:8000](http://127.0.0.1:8000);
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

```

**6.3. Liberar o Proxy no SELinux (CRÍTICO EM RHEL):**

```bash
sudo setsebool -P httpd_can_network_connect 1

```

**6.4. Validar e Recarregar NGINX:**

```bash
sudo nginx -t
sudo systemctl reload nginx

```

---

## 7. Como Testar a API

### Preparação do Teste Local:

```bash
sudo -u zabbix_api mkdir -p /zabbix_api/data/devices/unity/unityrio001/public/
sudo -u zabbix_api bash -c 'echo "{\"status\": \"ok\", \"cpu_utilization\": 45}" > /zabbix_api/data/devices/unity/unityrio001/public/done-unityrio001.json'

```

### Teste via Terminal:

```bash
curl -X GET "[http://127.0.0.1/devices/unity/unityrio001](http://127.0.0.1/devices/unity/unityrio001)" \
     -H "x-api-token: seu_token_super_secreto_123"

```

---

## 8. Coleta no Zabbix

### 8.1. Item Master

Criado no Template da família (ex: `Template Storage Unity`). Faz 1 única requisição HTTP.

* **Type:** `HTTP agent`
* **Key:** `api.get_json_data`
* **URL:** `http://ip_do_seu_nginx/devices/unity/{HOST.NAME}`
* **Headers:** Adicionar Name `x-api-token` e Value `{$API_TOKEN}`.
* **History storage period:** `0`.

### 8.2. Itens Dependentes

Criados para extrair valores do Item Master.

* **Type:** `Dependent item`
* **Master item:** `API: Obter dados do equipamento`
* **Preprocessing:**
* Step: `JSONPath`
* Parameters: `$.cpu_utilization`
* Ativar **Custom on fail** -> `Discard value`.
