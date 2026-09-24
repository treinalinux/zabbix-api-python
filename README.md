# Documentação de Integração: API Estática de Arquivos JSON para Zabbix

Esta documentação descreve a arquitetura e os passos de configuração para expor arquivos JSON estáticos locais de dispositivos (storages) para consumo no Zabbix.

A solução utiliza **Python (FastAPI)** operando sob o servidor de aplicação **Gunicorn com Uvicorn** em segundo plano, isolado em um **Virtual Environment (venv)** sob uma **Conta de Serviço**. A borda é protegida pelo **NGINX** atuando como proxy reverso com bloqueio de IP, e a autenticação ocorre via **Zabbix HTTP Agent** com cabeçalhos de segurança.

## 1. Arquitetura e Decisões Tecnológicas

* **FastAPI:** Framework moderno e extremamente rápido para criar a rota de leitura dos arquivos.
* **Gunicorn + Uvicorn:** O Gunicorn orquestra múltiplos "clones" (workers) do servidor ASGI Uvicorn. Isso garante alta disponibilidade (tolerância a falhas) e distribui a carga da API entre os núcleos da CPU.
* **Isolamento (VENV e System User):** O uso do `venv` impede conflitos com o Python do SO (crítico em RHEL/CentOS/AlmaLinux). A conta de serviço restringe as permissões da API, garantindo que, em caso de vulnerabilidade, o atacante não obtenha privilégios no servidor.

---

## 2. Preparação do Ambiente de Produção

Antes de escrever o código, vamos isolar a aplicação no sistema operacional.

### 2.1. Criar a Conta de Serviço

Crie um usuário de sistema (system account) que não possua diretório home ou permissão de login no terminal.

```bash
# Cria o usuário 'zabbix_api' sem acesso a shell
sudo useradd -r -s /bin/false zabbix_api
```

*(Nota: Certifique-se de que este usuário tenha permissão de **leitura** na pasta onde os JSONs são gerados: `chown -R zabbix_api:zabbix_api /caminho/absoluto/para/app/`)*

### 2.2. Criar e Ativar o Ambiente Virtual (VENV)

Navegue até o diretório onde sua API vai residir e crie o ambiente virtual.

```bash
cd /caminho/absoluto/para/app/no_api
python3 -m venv venv

# Ative o ambiente virtual
source venv/bin/activate
```

Com o `(venv)` ativado no terminal, instale as dependências. Elas ficarão isoladas nesta pasta.

```bash
pip install fastapi uvicorn gunicorn
```

---

## 3. Desenvolvimento da API (FastAPI)

A API atua como um leitor dinâmico. Ela recebe a requisição, monta o caminho até a pasta, valida o token de segurança e devolve o conteúdo do arquivo JSON.

**Código Fonte (`main.py`):**
Crie o arquivo principal no mesmo diretório onde você criou a pasta `venv`.

```python
import os
import json
from fastapi import FastAPI, HTTPException, Header

app = FastAPI()

# Caminho absoluto para a raiz dos arquivos gerados pelo sistema
BASE_PATH = "/caminho/absoluto/para/app/no_api/devices"

# Token de segurança esperado do Zabbix no cabeçalho da requisição
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

*(Após salvar o arquivo, você pode sair do ambiente virtual digitando `deactivate` no terminal).*

---

## 4. Configuração do Serviço em Segundo Plano (Systemd)

Agora vamos configurar o serviço para rodar com o nosso usuário dedicado e usar o executável do Gunicorn que está **dentro** do ambiente virtual.

**Criação do arquivo de serviço:**

```bash
sudo nano /etc/systemd/system/zabbix-api.service
```

**Configuração do Serviço (`zabbix-api.service`):**

```ini
[Unit]
Description=API Zabbix Leitora de JSON
After=network.target

[Service]
# Usa a conta de serviço criada no passo 2.1
User=zabbix_api
Group=zabbix_api

# Pasta onde está o main.py
WorkingDirectory=/caminho/absoluto/para/app/no_api

# Aponta para o Gunicorn DENTRO da pasta venv, orquestrando 4 workers
ExecStart=/caminho/absoluto/para/app/no_api/venv/bin/gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 127.0.0.1:8000

Restart=always

[Install]
WantedBy=multi-user.target
```

**Ativação do Serviço:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable zabbix-api
sudo systemctl start zabbix-api

# Para checar se iniciou corretamente:
# sudo systemctl status zabbix-api
```

---

## 5. Segurança e Proxy Reverso (NGINX)

O NGINX intercepta as requisições na porta 80, verifica a origem do IP (camada de rede) e encaminha o tráfego autorizado para a API Python na porta 8000. 

**Configuração do Bloco de Servidor (`/etc/nginx/sites-available/default`):**

```nginx
server {
    listen 80;
    server_name seu-servidor.com;

    # Rota da aplicação principal já existente (Mantém intacta)
    location / {
        # ... configurações existentes da sua outra aplicação ...
    }

    # Rota exclusiva para consumo do Zabbix (Protegida)
    location /devices/ {
        # 1. Permite acesso apenas dos IPs autorizados (Zabbix Server/Proxy)
        allow 192.168.1.50; 
        
        # 2. Bloqueia qualquer outro IP com Erro 403 (Forbidden)
        deny all;

        # 3. Encaminha a requisição autorizada para a API Python (porta 8000)
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Aplicação das Regras (Zero Downtime):**

```bash
sudo nginx -t
sudo nginx -s reload
```

---

## 6. Configuração de Coleta no Zabbix

### 6.1. Item Master (Coletor Principal)

Criado no Template de cada família de equipamentos (ex: `Template Storage Unity`).

* **Name:** `API: Obter dados do equipamento`
* **Type:** `HTTP agent`
* **Key:** `api.get_json_data`
* **URL:** `http://ip_do_seu_nginx/devices/unity/{HOST.NAME}`
* **Type of information:** `Text`
* **Update interval:** `5m`
* **History storage period:** `0` (Impede o banco de dados de armazenar os JSONs brutos)
* **Headers:**
  * Name: `x-api-token`
  * Value: `{$API_TOKEN}` *(Macro contendo o valor configurado na API)*

### 6.2. Itens Dependentes (Métricas Específicas)

Para cada métrica, cria-se um item dependente do Item Master.

* **Name:** `CPU Utilization` 
* **Type:** `Dependent item`
* **Key:** `unity.cpu.utilization`
* **Master item:** `API: Obter dados do equipamento`
* **Type of information:** `Numeric (float)` ou `Text`

**Aba Preprocessing:**
* **Step:** `JSONPath`
* **Parameters:** `$.cpu_utilization` 
* **Custom on fail:** Ative e selecione `Discard value`.

---

## 7. Testando a API 

> **Atenção:** Como configuramos bloqueio de IP no NGINX, os testes externos só funcionarão se o seu IP estiver na lista de `allow` do NGINX, ou se o teste for executado diretamente do servidor (localhost).

### Testando via terminal (cURL)
```bash
curl -X GET "http://ip_do_seu_nginx/devices/unity/unityrio001" \
     -H "x-api-token: seu_token_super_secreto_123"
```

### Testando via Postman / Insomnia
1. Crie uma requisição **GET**.
2. URL: `http://ip_do_seu_nginx/devices/unity/unityrio001`
3. Aba **Headers**:
   * Chave: `x-api-token`
   * Valor: `seu_token_super_secreto_123`
4. Envie a requisição para validar o retorno do JSON.%
