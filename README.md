
# 📦 BOX - Simplifique Suas Integrações com um Gateway de APIs Universal

**BOX** é um **API Gateway** construído em **ASP.NET Core** baseado no YARP, com foco em **orquestração**, **segurança**, **resiliência**, e **configuração dinâmica** via JSON. Ele centraliza o acesso a múltiplas APIs internas e se comunica com elas via rede Docker interna. Todo o comportamento do gateway é controlado pelo `service.json`.

---

## 🚀 Funcionalidades

- 🔁 Proxy Reverso  
- 🔐 Autenticação JWT  
- 🔁 Retry automático  
- 🛡️ Rate Limiting  
- ⚠️ Tratamento centralizado de erros  
- 🧠 Arquitetura extensível  
- ⚖️ Load Balancing  

---

## 🧩 Exemplo Completo de `service.json`

```json
{
  // URL para autenticação do usuário
  "AuthOrigin": "http://localhost:3000/verify-user",

  // Políticas de Rate Limiting por serviço
  "RateLimit": {
    "serviceUserLimit": {
      // Limite de requisições permitidas
      "PermitLimit": 10,
      // Tipo da janela de tempo (horas)
      "WindowTypeTime": "HOURS",
      // Duração da janela
      "WindowTime": 2,
      // Ordem de processamento da fila
      "QueueProcessingOrder": "OLDEST_FIRST",
      // Tamanho máximo da fila (0 = sem fila)
      "QueueLimit": 0
    },
    "servicePostLimit": {
      "PermitLimit": 10,
      "WindowTypeTime": "HOURS",
      "WindowTime": 2,
      "QueueProcessingOrder": "OLDEST_FIRST",
      "QueueLimit": 0
    },
    "serviceTodoLimit": {
      "PermitLimit": 10,
      "WindowTypeTime": "HOURS",
      "WindowTime": 2,
      "QueueProcessingOrder": "OLDEST_FIRST",
      "QueueLimit": 0
    }
  },

  // Política de retry automática para falhas de requisição
  "HttpRetry": {
    // Tempo entre as tentativas
    "DelayTime": 5,
    // Unidade do tempo de delay
    "DelayTypeTime": "SECONDS",
    // Número máximo de tentativas
    "MaxRetryAttempts": 6
  },

  // Configuração do Proxy Reverso
  "ReverseProxy": {
    "Routes": {
      // Rota para o serviço de usuários
      "serviceUser": {
        // Nome do cluster
        "ClusterId": "serviceUser",
        // Política de autenticação aplicada
        "AuthorizationPolicy": "AuthPolicy",
        // Política de rate limit aplicada
        "RateLimiterPolicy": "serviceUserLimit",
        // Padrão de URL atendido
        "Match": {
          "Path": "/users/{**catch-all}"
        }
      },
      // Rota para o serviço de posts
      "servicePost": {
        "ClusterId": "servicePost",
        "AuthorizationPolicy": "AuthPolicy",
        "RateLimiterPolicy": "servicePostLimit",
        "Match": {
          "Path": "/posts/{**catch-all}"
        }
      },
      // Rota para o serviço de tarefas (todos)
      "serviceTodo": {
        "ClusterId": "serviceTodo",
        "AuthorizationPolicy": "AuthPolicy",
        "RateLimiterPolicy": "serviceTodoLimit",
        "Match": {
          "Path": "/todos/{**catch-all}"
        }
      }
    },

    // Definições dos clusters/destinos
    "Clusters": {
      "serviceUser": {
        "Destinations": {
          // Destino único para o cluster de usuários
          "origin1": {
            "Address": "https://jsonplaceholder.typicode.com/"
          }
        }
      },
      "servicePost": {
        "Destinations": {
          "origin1": {
            "Address": "https://jsonplaceholder.typicode.com/"
          }
        }
      },
      "serviceTodo": {
        // Política de balanceamento de carga
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          // Múltiplos destinos para balanceamento
          "origin1": {
            "Address": "https://jsonplaceholder.typicode.com/"
          },
          "origin2": {
            "Address": "https://jsonplaceholder.typicode.com/"
          }
        }
      }
    }
  }
}
```

---

## 🐳 Exemplo Completo de `docker-compose.yml`

```yaml
version: '3.8'
services:

  gateway:
    image: isaiasdevback/box:latest           # Imagem do Gateway publicada no Docker Hub
    ports:
      - "8080:8080"
    volumes:
      - ./service.yaml:/app/service.yaml  # Monta o arquivo de config
    depends_on:
      - service-post
      - service-user
      - service-cep
      - service-auth
      - service-stats
    networks:
      - internal

  service-post:
    image: seuusuario/service-post:latest
    networks:
      - internal

  service-user:
    image: seuusuario/service-user:latest
    networks:
      - internal

  service-cep:
    image: seuusuario/service-cep:latest
    networks:
      - internal

  service-auth:
    image: seuusuario/service-auth:latest
    networks:
      - internal

  service-stats:
    image: seuusuario/service-stats:latest
    networks:
      - internal

networks:
  internal:
    driver: bridge
```

---

## ▶️ Como executar

1. Tenha o Docker instalado.
2. Crie um arquivo `service.yaml` com base no exemplo acima.
3. Crie um `docker-compose.yml` com os serviços internos.
4. Rode:

```bash
docker-compose up -d
```

O gateway ficará disponível em:

```
http://localhost:8080
```

---

## 🔐 Autenticação JWT

- Cada requisição com JWT precisa de um header:

```http
Authorization: Bearer <seu_token_aqui>
```

- O gateway valida o usuário chamando o endpoint que foi definido no serviço `AuthOrigin`.
- O serviço definido em `auth`, deve retornar um json que contenha a propriedade `{ "isAuthenticated": true or false }`, caso seja `true` ele gera um Bearer Token e autentica.
    
---

## 🔁 Retry com Polly

- Configurado via `httpRetry` no `service.yaml`
- Em caso de erro temporário (timeout, 5xx), o gateway tentará novamente
- Respeita o número máximo de tentativas e tempo entre elas

---

## 🛡️ Rate Limiting

- Limita o número de requisições por janela de tempo.
- Requisições acima do limite recebem código HTTP 429.

---

## ⚠️ Tratamento de Erros

- Todos os erros passam por middlewares centralizados
- Erros da API retornam mensagens padronizadas:
  
```json
{
  "origin": "http://service.m1:5000",
  "endpoint": "/users",
  "statusCode": 500,
  "data": { /* resposta de erro do seu serviço */ }
}
```

---

## 🧪 Exemplo de Requisição

```bash
curl http://localhost:8080/users   -H "Authorization: Bearer SEU_TOKEN_JWT"
```

---

## 📌 Funcionalidades Futuras

Essas são as próximas melhorias previstas para o **BOX - API Gateway**:

- 🧰 **Cache Inteligente**  
  Implementação de cache para respostas frequentes, com configurações dinâmicas e invalidação automática.

- 🪵 **Geração de Arquivo de Log**  
  Logs estruturados para requisições, respostas, erros e métricas de uso em formato `.log` para facilitar auditoria e análise.

---

## 📬 Contato

Desenvolvido por Isaías Vasconcelos.  
Sinta-se livre para abrir issues ou sugerir melhorias.

---

## 🧠 Por que BOX?

> Porque tudo cabe aqui dentro.  
> Centralize, controle e orquestre suas APIs com segurança e inteligência.
