# Documentação de Integração: API Estática de Arquivos JSON para Zabbix

Esta documentação descreve a arquitetura e os passos de configuração para expor arquivos JSON estáticos locais de dispositivos (storages) para consumo no Zabbix.

A solução utiliza **Python (FastAPI)** operando sob o servidor de aplicação **Gunicorn com Uvicorn** em segundo plano, **NGINX** como proxy reverso com bloqueio de IP, e o **Zabbix HTTP Agent** com cabeçalhos de autenticação.

## 1. Arquitetura e Decisão Tecnológica

* **FastAPI:** Framework moderno e extremamente rápido para criar a rota de leitura dos arquivos estáticos.

* **Por que Gunicorn + Uvicorn para Produção?**
  O *Uvicorn* isolado é um servidor web ASGI ultrarrápido, porém roda em um único processo (single-core). Para um ambiente de **produção tolerante a falhas**, utilizamos o *Gunicorn* como "gerente de processos". O Gunicorn orquestra múltiplos "clones" (workers) do Uvicorn, distribuindo a carga entre os núcleos da CPU do servidor e garantindo que, se um processo travar por qualquer motivo, um novo seja iniciado imediatamente, garantindo estabilidade e alta disponibilidade.

## 2. Desenvolvimento da API (FastAPI)

A API atua como um leitor dinâmico em tempo real. Ela recebe a requisição, monta o caminho até a pasta do dispositivo, valida o token de segurança e devolve o conteúdo do arquivo JSON.

**Dependências:**

```bash
pip install fastapi uvicorn gunicorn
```

**Código Fonte (`main.py`):**
Crie o arquivo principal da aplicação. Certifique-se de ajustar o diretório apontado em `BASE_PATH`.

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

## 3. Configuração do Serviço em Segundo Plano (Systemd)

Para garantir que a API inicie automaticamente com o sistema operacional e permaneça em execução, configuramos um serviço no Linux gerenciado pelo Systemd utilizando o Gunicorn.

**Criação do arquivo de serviço:**

```bash
sudo nano /etc/systemd/system/zabbix-api.service
```

**Configuração do Serviço (`zabbix-api.service`):**
Substitua `seu_usuario` e `/caminho/absoluto/onde/esta/o/main_py` pelos valores reais do seu ambiente. *(Dica: 4 workers são suficientes para o consumo do Zabbix, mas a regra geral é: Número de Cores da CPU x 2 + 1).*

```ini
[Unit]
Description=API Zabbix Leitora de JSON
After=network.target

[Service]
User=seu_usuario
WorkingDirectory=/caminho/absoluto/onde/esta/o/main_py

# Executa o Gunicorn gerenciando 4 workers do Uvicorn
ExecStart=/usr/local/bin/gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 127.0.0.1:8000
Restart=always

[Install]
WantedBy=multi-user.target
```

*(Nota: Se o Gunicorn estiver instalado em outro caminho, valide usando o comando `which gunicorn` e altere o parâmetro `ExecStart`)*.

**Ativação do Serviço:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable zabbix-api
sudo systemctl start zabbix-api
```

## 4. Segurança e Proxy Reverso (NGINX)

O NGINX intercepta as requisições na porta 80, verifica a origem do IP (camada de rede) e encaminha o tráfego autorizado para a API Python na porta 8000. Isso é feito de forma isolada, sem afetar outras aplicações rodando no mesmo servidor.

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
        # Adicione também o seu IP se quiser testar de fora do servidor
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

## 5. Configuração de Coleta no Zabbix

A coleta é realizada através de um item mestre para otimizar as requisições e itens dependentes para gerar o histórico e gráficos, reduzindo o tráfego de rede e o processamento no servidor monitorado.

### 5.1. Item Master (Coletor Principal)

Este item é responsável apenas por bater na API e baixar o JSON bruto temporariamente em memória. Deve ser criado no Template de cada família de equipamentos (ex: `Template Storage Unity`).

* **Name:** `API: Obter dados do equipamento`
* **Type:** `HTTP agent`
* **Key:** `api.get_json_data`
* **URL:** `http://ip_do_seu_nginx/devices/unity/{HOST.NAME}`
* **Type of information:** `Text`
* **Update interval:** `5m` (Conforme necessidade de monitoramento)
* **History storage period:** `0` (Importante: evita armazenamento do JSON inteiro no banco do Zabbix)
* **Headers:**
  * Name: `x-api-token`
  * Value: `{$API_TOKEN}` *(Macro definida globalmente ou no template contendo o valor do token configurado na API)*

### 5.2. Itens Dependentes (Métricas Específicas)

Para cada valor numérico ou de texto contido no JSON (CPU, disco, status, etc.), cria-se um item dependente atrelado ao Item Master.

* **Name:** `CPU Utilization` (Exemplo)
* **Type:** `Dependent item`
* **Key:** `unity.cpu.utilization`
* **Master item:** `API: Obter dados do equipamento`
* **Type of information:** `Numeric (float)` ou `Text` (conforme o tipo do dado)

**Aba Preprocessing do Item Dependente:**

* **Step:** `JSONPath`
* **Parameters:** `$.cpu_utilization` (Exemplo: Caminho exato da chave de métrica dentro do arquivo JSON gerado).
* **Custom on fail:** Ative esta opção e selecione `Discard value`. Isso previne falsos alertas e erros de leitura caso uma chave específica falte pontualmente no arquivo JSON de origem.

## 6. Testando a API (Curl, Postman e Insomnia)

Para garantir que a comunicação e a autenticação estão funcionando antes de configurar o Zabbix, você pode simular requisições. 

> **Atenção:** Como configuramos bloqueio de IP no NGINX, os testes externos só funcionarão se o seu IP atual estiver na lista de `allow` do NGINX, ou se você estiver rodando o teste diretamente de dentro do servidor (via localhost).

### Testando via terminal (cURL)
Use o parâmetro `-H` para passar o cabeçalho de autenticação:

```bash
curl -X GET "http://ip_do_seu_nginx/devices/unity/unityrio001" \
     -H "x-api-token: seu_token_super_secreto_123"
```

### Testando via Postman
1. Crie uma nova requisição clicando em **"+"** ou **"New"**.
2. Altere o método HTTP para **GET**.
3. Na barra de URL, insira: `http://ip_do_seu_nginx/devices/unity/unityrio001`
4. Abaixo da barra de URL, clique na aba **Headers**.
5. Adicione uma nova linha:
   * **Key:** `x-api-token`
   * **Value:** `seu_token_super_secreto_123`
6. Clique em **Send**. O JSON deve aparecer na aba de "Body" inferior.

### Testando via Insomnia
1. Pressione **Ctrl+N** (ou Cmd+N) para criar uma nova requisição (New Request).
2. Dê um nome, defina o método como **GET** e confirme.
3. Na barra superior, insira a URL: `http://ip_do_seu_nginx/devices/unity/unityrio001`
4. Na aba **Headers** logo abaixo da URL, adicione:
   * **Header:** `x-api-token`
   * **Value:** `seu_token_super_secreto_123`
5. Clique em **Send** para visualizar o JSON de resposta no painel direito.%
