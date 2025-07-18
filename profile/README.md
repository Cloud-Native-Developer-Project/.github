# Arquitectura AWS Cloud Native para Aplicación de CV

## Resumen Ejecutivo

Esta aplicación será completamente serverless y cloud-native, aprovechando servicios AWS que son fundamentales para la certificación Developer Associate, como Lambda, API Gateway, DynamoDB, S3, y servicios de integración.

## Arquitectura General

```
Frontend (React/Vue) → CloudFront → API Gateway → Lambda Functions → DynamoDB/S3
                                                     ↓
                                              SQS → Step Functions
                                                     ↓
                                              EventBridge → Lambda
```

## Módulos y Servicios AWS Recomendados

### 1. **Módulo de Autenticación**
**Servicio AWS:** Amazon Cognito (en lugar de Keycloak)

**Justificación:** Cognito es el servicio nativo de AWS para autenticación y autorización, completamente serverless y altamente escalable.

**Componentes:**
- **Cognito User Pool:** Gestión de usuarios y autenticación
- **Cognito Identity Pool:** Autorización y acceso a recursos AWS
- **Lambda Triggers:** Para lógica personalizada (pre-signup, post-confirmation, etc.)

**Implementación:**
```javascript
// Ejemplo de configuración con AWS SDK
const cognitoIdentityServiceProvider = new AWS.CognitoIdentityServiceProvider();
```

### 2. **Módulo de Subscripciones**
**Servicio AWS:** API Gateway + Lambda + DynamoDB + EventBridge

**Justificación:** Implementación serverless que integra con servicios de pago como Stripe.

**Componentes:**
- **API Gateway:** Endpoints para gestión de subscripciones
- **Lambda Functions:** Lógica de negocio para pagos
- **DynamoDB:** Almacenamiento de datos de subscripción
- **EventBridge:** Eventos para renovaciones y notificaciones
- **SES:** Envío de emails de confirmación

**Flujo:**
```
Webhook Stripe → API Gateway → Lambda → DynamoDB → EventBridge → Lambda (Notifications)
```

### 3. **Módulo de Extracción de Datos**
**Servicio AWS:** S3 + Lambda + Textract

**Justificación:** Textract es el servicio AWS para extracción de texto de documentos.

**Componentes:**
- **S3:** Almacenamiento de CVs subidos
- **Lambda:** Procesamiento y orquestación
- **Textract:** Extracción de texto de PDFs/imágenes
- **SQS:** Cola para procesamiento asíncrono

**Flujo:**
```
Frontend → S3 Upload → Lambda Trigger → Textract → Lambda Process → DynamoDB
```

### 4. **Módulo de Formateo/Optimización**
**Servicio AWS:** Lambda + SQS + DynamoDB

**Justificación:** Procesamiento serverless con cola para manejar latencia de APIs externas.

**Componentes:**
- **Lambda:** Integración con OpenAI API
- **SQS:** Cola para procesamiento asíncrono
- **DynamoDB:** Cache de respuestas de IA
- **Parameter Store:** Almacenamiento seguro de API keys

**Implementación:**
```javascript
// Lambda function para formateo
exports.handler = async (event) => {
    const openai = new OpenAI({
        apiKey: await getParameter('/cv-app/openai-key')
    });
    
    const completion = await openai.chat.completions.create({
        messages: [{ role: "user", content: event.cvText }],
        model: "gpt-3.5-turbo",
    });
    
    return {
        statusCode: 200,
        body: JSON.stringify(completion.choices[0].message.content)
    };
};
```

### 5. **Módulo de Generación de PDF**
**Servicio AWS:** Lambda + S3

**Justificación:** Generación serverless de PDFs con almacenamiento en S3.

**Componentes:**
- **Lambda:** Generación de PDF con bibliotecas como Puppeteer
- **S3:** Almacenamiento de PDFs generados
- **CloudFront:** CDN para entrega rápida de PDFs

**Layer personalizado:**
```bash
# Crear layer para Puppeteer
npm install puppeteer-core
zip -r puppeteer-layer.zip node_modules/
```

### 6. **Módulo de Modificación del CV**
**Servicio AWS:** API Gateway + Lambda + DynamoDB

**Justificación:** APIs REST serverless para CRUD operations.

**Componentes:**
- **API Gateway:** Endpoints REST para modificaciones
- **Lambda:** Lógica de negocio para CRUD
- **DynamoDB:** Almacenamiento de datos del CV
- **DAX:** Cache para DynamoDB (opcional)

### 7. **Módulo de Persistencia**
**Servicio AWS:** DynamoDB + ElastiCache + S3

**Justificación:** Combinación de NoSQL, cache y almacenamiento de objetos.

**Componentes:**
- **DynamoDB:** Base de datos principal
- **ElastiCache Redis:** Cache de sesiones y datos frecuentes
- **S3:** Almacenamiento de archivos (CVs, PDFs)
- **CloudWatch:** Monitoreo y logs

## Arquitectura Detallada por Capa

### **Frontend**
```
React/Vue.js → AWS Amplify → CloudFront → S3 (Static Hosting)
```

### **API Layer**
```
API Gateway → Lambda Functions → DynamoDB/S3
```

### **Processing Layer**
```
SQS → Lambda → Step Functions → External APIs
```

### **Storage Layer**
```
DynamoDB (NoSQL) + S3 (Object Storage) + ElastiCache (Cache)
```

## Flujo de Trabajo Completo

### 1. **Subida de CV Existente**
```
Frontend → S3 → Lambda (Trigger) → Textract → Lambda (Process) → DynamoDB
```

### 2. **Formateo con IA**
```
Frontend → API Gateway → Lambda → SQS → Lambda (OpenAI) → DynamoDB
```

### 3. **Generación de PDF**
```
Frontend → API Gateway → Lambda (PDF Gen) → S3 → CloudFront
```

### 4. **Modificación de CV**
```
Frontend → API Gateway → Lambda (CRUD) → DynamoDB
```

## Configuración de Infraestructura como Código

### **AWS CDK (Recomendado para Developer Associate)**

```typescript
// Ejemplo de stack principal
export class CvAppStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // DynamoDB Table
    const cvTable = new Table(this, 'CVTable', {
      tableName: 'cv-data',
      partitionKey: { name: 'userId', type: AttributeType.STRING },
      sortKey: { name: 'cvId', type: AttributeType.STRING },
      billingMode: BillingMode.PAY_PER_REQUEST,
    });

    // Lambda Functions
    const formatCvFunction = new Function(this, 'FormatCvFunction', {
      runtime: Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: Code.fromAsset('lambda/format-cv'),
      environment: {
        TABLE_NAME: cvTable.tableName,
      },
    });

    // API Gateway
    const api = new RestApi(this, 'CvApi', {
      restApiName: 'CV Service',
      description: 'API for CV processing',
    });

    const cvResource = api.root.addResource('cv');
    cvResource.addMethod('POST', new LambdaIntegration(formatCvFunction));
  }
}
```

## Servicios de Monitoreo y Observabilidad

### **CloudWatch**
- Logs de Lambda functions
- Métricas de API Gateway
- Dashboards personalizados

### **X-Ray**
- Trazabilidad de requests
- Performance insights
- Error tracking

### **AWS Config**
- Compliance y configuración
- Auditoría de cambios

## Seguridad

### **IAM Roles y Policies**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/cv-data"
    }
  ]
}
```

### **Cognito Security**
- MFA habilitado
- Password policies
- JWT token validation

## Estimación de Costos

### **Tier Gratuito (Primeros 12 meses)**
- Lambda: 1M requests/mes
- DynamoDB: 25GB storage
- S3: 5GB storage
- CloudFront: 50GB transfer

### **Costos Estimados Mensuales (después del tier gratuito)**
- Lambda: $0.20 por 1M requests
- DynamoDB: $0.25 por GB/mes
- S3: $0.023 por GB/mes
- API Gateway: $3.50 por 1M requests

## Pasos de Implementación

### **Fase 1: Configuración Básica**
1. Configurar AWS CLI y CDK
2. Crear Cognito User Pool
3. Implementar autenticación básica
4. Crear API Gateway con Lambda

### **Fase 2: Funcionalidad Core**
1. Implementar subida a S3
2. Configurar Textract para extracción
3. Integrar OpenAI API
4. Crear generación de PDF

### **Fase 3: Funcionalidades Avanzadas**
1. Implementar sistema de subscripciones
2. Añadir cache con ElastiCache
3. Configurar monitoreo con CloudWatch
4. Implementar CI/CD pipeline

### **Fase 4: Optimización y Producción**
1. Configurar CloudFront
2. Implementar X-Ray tracing
3. Añadir tests automatizados
4. Configurar alertas y monitoring

## Beneficios para la Certificación Developer Associate

### **Servicios AWS Cubiertos:**
- ✅ Lambda (Serverless computing)
- ✅ API Gateway (API management)
- ✅ DynamoDB (NoSQL database)
- ✅ S3 (Object storage)
- ✅ Cognito (Authentication)
- ✅ CloudFormation/CDK (Infrastructure as Code)
- ✅ CloudWatch (Monitoring)
- ✅ SQS (Message queuing)
- ✅ Step Functions (Workflow orchestration)

### **Conceptos Clave:**
- Serverless architecture
- Event-driven programming
- Infrastructure as Code
- Security best practices
- Monitoring and logging
- Cost optimization

## Recursos de Aprendizaje

### **Documentación AWS:**
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/dynamodb/)
- [API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/)

### **Tutoriales Prácticos:**
- AWS Workshops
- AWS Samples en GitHub
- Hands-on labs

---
