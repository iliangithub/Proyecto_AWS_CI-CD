# Proyecto AWS CI/CD


## 1.0 Beanstalk

> [!IMPORTANT]
> New accounts only support launch templates
Starting on October 1, 2024, Amazon EC2 Auto Scaling will no longer support the creation of launch configurations for new accounts. Existing environments will not be impacted. For more information about other situations that are impacted, including temporary option settings required for new accounts, refer to Launch templates  in the Elastic Beanstalk Developer Guide.
>


## 1.1 Crear Pares-Clave.

- Name: `clavebeanstalk`
- Key Pair Type: `RSA`
- Private Key File Format: `.pem`

## 1.2 Crear aplicación.
### Paso 1. Configuración del entorno

- Escogemos `Entorno de servidor web`.
- Application name: `gamma`.
- Enviroment name: `gamma-prod11`.
- Domain: `gamma-prod11` y CHECK AVAILABILITY.
- Tipo de plataforma: `Plataforma administrada`.
- Plataforma: `TomCat` (la platform branch Tomcat 10 correto 21 y platform version 5.3.0)
- Application code: `Sample application`.
- Custom configuration.

### Paso 2. Configuración del acceso al servicio

- Service Role: `Use existing service role`, `aws-elasticbeanstalk-service-role`
- EC2 key pair: `clavebeantstalk.pem`.
- EC2 instance profile: 
  
### Paso 3. Configuración de redes, bases de datos y etiquetas

### Paso 4. Configuración del escalado y del tráfico de instancias

### Paso 5. Configuración de actualizaciones, monitoreo y registros
