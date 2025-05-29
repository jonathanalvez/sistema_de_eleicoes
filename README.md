## Criação do Usuário Administrador Helios

Após rodar as migrações, é necessário criar um usuário administrador para acessar o sistema Helios. O Helios utiliza um sistema de autenticação próprio, diferente do superusuário padrão do Django.

Execute o seguinte comando para abrir o shell do Django:

```
python manage.py shell
```

E dentro do shell, execute:

```python
from helios_auth.models import User
user = User(user_type='password', user_id='admin', name='Administrador', info={'password': '123'})
user.save()
```

Agora você pode acessar o sistema com:

- **Usuário:** `admin`
- **Senha:** `123`

Altere a senha após o primeiro login para maior segurança.

## Configuração do Upload de Eleitores

Para que o upload de eleitores funcione corretamente, é necessário configurar o Celery e o RabbitMQ. Siga os passos abaixo:

### Por que precisamos do Celery e RabbitMQ?

O sistema Helios utiliza o Celery e o RabbitMQ para processar tarefas em segundo plano, especialmente durante o upload de eleitores. Isso é necessário porque:

1. **Processamento Assíncrono**: Quando você faz upload de uma lista de eleitores, o sistema precisa:

   - Processar o arquivo CSV/Excel
   - Validar os dados de cada eleitor
   - Gerar credenciais únicas
   - Enviar emails para cada eleitor

   Se isso fosse feito diretamente no servidor web, você teria que esperar todo o processo terminar antes de receber uma resposta, o que poderia levar vários minutos para listas grandes.

2. **Confiabilidade**: O Celery garante que:

   - As tarefas não sejam perdidas se houver algum erro
   - Os emails sejam enviados mesmo se o navegador for fechado
   - O processo continue mesmo se houver falhas temporárias

3. **Escalabilidade**: O sistema pode:
   - Processar múltiplos uploads simultaneamente
   - Distribuir o trabalho entre vários workers
   - Lidar com grandes volumes de eleitores

### Como Celery e RabbitMQ trabalham juntos?

O Celery e o RabbitMQ trabalham em conjunto da seguinte forma:

1. **Celery**: É o sistema de processamento de tarefas em segundo plano que:

   - Recebe as tarefas do Django
   - Gerencia os workers (processos que executam as tarefas)
   - Controla o fluxo de trabalho

2. **RabbitMQ**: É o "broker" (intermediário) que:
   - Armazena as mensagens/tarefas em uma fila
   - Garante que nenhuma mensagem seja perdida
   - Distribui as tarefas entre os workers do Celery

O fluxo de trabalho é assim:

1. O Django envia uma tarefa para o RabbitMQ
2. O RabbitMQ armazena a tarefa em uma fila
3. O Celery pega a tarefa da fila
4. O Celery distribui a tarefa para um worker disponível
5. O worker processa a tarefa (por exemplo, envia emails)

É por isso que precisamos dos dois serviços: o RabbitMQ para gerenciar as filas de mensagens e o Celery para processar essas mensagens.

### 1. Instalação do RabbitMQ

No Windows, você tem duas opções:

1. Baixar o instalador do RabbitMQ em: https://www.rabbitmq.com/install-windows.html
2. Ou usar o Chocolatey:
   ```
   choco install rabbitmq
   ```

Após a instalação, o serviço do RabbitMQ deve iniciar automaticamente. Você pode verificar se está rodando no Gerenciador de Serviços do Windows.

### 2. Instalação das Dependências

Instale as dependências necessárias:

```bash
pip install celery
```

### 3. Iniciando os Serviços

Você precisa manter dois terminais abertos:

1. Terminal 1 - Servidor Django:

```bash
python manage.py runserver
```

2. Terminal 2 - Worker do Celery:

No Windows, use o comando abaixo (com o parâmetro -b):

```powershell
celery -A helios -b amqp://guest:guest@localhost:5672// worker -l info --pool=solo
```

Se estiver em outro sistema operacional, o comando padrão é:

```bash
celery -A helios worker -l info
```

### Verificando se está tudo funcionando

1. **Verificar RabbitMQ**:

   - Abra o Gerenciador de Serviços do Windows
   - Procure por "RabbitMQ Server"
   - Deve estar "Em execução"
   - Se não estiver, clique com botão direito e selecione "Iniciar"

2. **Verificar Conexão**:

   - Abra o navegador e acesse: http://localhost:15672
   - Use as credenciais padrão:
     - Usuário: `guest`
     - Senha: `guest`
   - Se conseguir acessar, o RabbitMQ está funcionando corretamente

3. **Testar Celery**:
   - No terminal onde o Celery está rodando, você deve ver mensagens como:
     ```
     [INFO/MainProcess] Connected to amqp://guest:**@localhost:5672//
     [INFO/MainProcess] mingle: searching for neighbors
     [INFO/MainProcess] mingle: all alone
     [INFO/MainProcess] celery@seucomputador ready.
     ```
   - Se aparecer algum erro, verifique:
     - Se o RabbitMQ está rodando
     - Se as portas 5672 (AMQP) e 15672 (gerenciamento) estão abertas
     - Se não há firewall bloqueando as conexões

### Solução de Problemas Comuns

1. **Erro de Conexão**:

   - Verifique se o RabbitMQ está rodando
   - Tente reiniciar o serviço do RabbitMQ
   - Verifique se as credenciais no `settings.py` estão corretas:
     ```python
     BROKER_URL = 'amqp://guest:guest@localhost:5672//'
     ```

2. **Erro de Permissão**:

   - Execute o terminal como administrador
   - Verifique se o usuário tem permissão para acessar o RabbitMQ

3. **Erro de Porta**:
   - Verifique se a porta 5672 está disponível
   - Use o comando `netstat -an | findstr 5672` para ver se a porta está em uso

Após seguir todos esses passos, o upload de eleitores deve funcionar corretamente, processando o arquivo em segundo plano e enviando os emails para os eleitores.
