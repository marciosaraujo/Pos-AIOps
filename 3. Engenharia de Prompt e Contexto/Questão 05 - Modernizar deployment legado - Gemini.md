## Prompt
```
[BEFORE] 
Aqui está um manifest de Deployment do Kubernetes legado (`chronos-api`) de um ambiente de produção. Ele apresenta vários antipadrões graves: usa a tag de imagem `latest`, possui apenas 1 réplica configurada (ponto único de falha), expõe credenciais sensíveis (DB_PASSWORD e JWT_SECRET) em texto plano no array `env`, e não possui configurações básicas de resiliência e segurança, como limits/requests, health checks e security context.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"

[AFTER] 
Preciso da versão modernizada deste manifest, alinhada com as melhores práticas para ambientes enterprise. O resultado deve ter alta disponibilidade (mínimo de 3 réplicas), utilizar uma imagem versionada (ex: `v2.1.0`), carregar as secrets de forma segura usando `secretKeyRef` (assumindo um Secret chamado `chronos-api-secrets`), definir `resources` (requests e limits dimensionados para uma API padrão), incluir `livenessProbe` e `readinessProbe` baseados na porta 8080, e aplicar um `securityContext` rigoroso no container (não-root, sistema de arquivos read-only, e sem escalação de privilégios).

[BRIDGE] 
Reescreva o código YAML completo aplicando todas essas exigências. Inclua comentários curtos e diretos no próprio código explicando cada nova adição de segurança e resiliência. Não gere explicações fora do bloco de código YAML, entregue apenas o manifest pronto para ser aplicado.
```
---
## Modelo
```
Gemini 3.1 Pro
```
---
## Output
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
spec:
  # Alta disponibilidade garantida
  replicas: 3
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      # Pod Security Context básico para restringir o pod inteiro
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      containers:
      - name: api
        # Imagem versionada evitando o uso de 'latest'
        image: chronos-api:v2.1.0
        ports:
        - containerPort: 8080
          name: http
        
        # Referência segura às credenciais através de um K8s Secret externo
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: jwt-secret

        # Dimensionamento de recursos para scheduling eficiente e proteção do Node
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

        # Security Context no nível do container limitando permissões de SO
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL
        
        # Probes para garantir que o tráfego só chegue a pods saudáveis
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 15
          periodSeconds: 20
```
---
## Justificativa

O framework B-A-B demonstra como o prompt estrutura a refatoração do manifest: o Before estabelece o contexto inicial ao apresentar o código legado e seus antipadrões técnicos; o After define o estado final desejado, listando as exigências de modernização para um ambiente enterprise (como alta disponibilidade e segurança); e a Bridge atua como a instrução acionável que conecta os dois pontos, direcionando a inteligência artificial a gerar o código YAML reescrito e documentado de forma direta e sem explicações adicionais.
