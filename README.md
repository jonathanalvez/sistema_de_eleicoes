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
