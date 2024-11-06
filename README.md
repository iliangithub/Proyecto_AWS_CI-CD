# Proyecto AWS CI/CD


## 1.0 Beanstalk

> [!IMPORTANT]
> New accounts only support launch templates
Starting on October 1, 2024, Amazon EC2 Auto Scaling will no longer support the creation of launch configurations for new accounts. Existing environments will not be impacted. For more information about other situations that are impacted, including temporary option settings required for new accounts, refer to Launch templates  in the Elastic Beanstalk Developer Guide.
>


## 1.1 Crear Pares-Clave y Rol.

### Pares-Clave.
- Name: `clavebeanstalk`
- Key Pair Type: `RSA`
- Private Key File Format: `.pem`

### Crear rol IAM.

- Tipo de entidad de confianza: `Servicio AWS`
- Servicio o caso de uso: `EC2`.

> [!TIP]
> Al crear un rol IAM para Elastic Beanstalk, debes seleccionar EC2 como caso de uso porque Elastic Beanstalk normalmente administra instancias de EC2 para ejecutar tu aplicación. Esto garantiza que el rol tendrá los permisos correctos para realizar tareas como iniciar, detener y administrar esas instancias EC2 según sea necesario.
>

> [!TIP]
>El "servicio o caso de uso" al crear un rol IAM en AWS permite a IAM aplicar configuraciones y permisos específicos según el contexto en que el rol será usado. Esencialmente, AWS quiere saber qué servicios necesitarán permisos para actuar en tu cuenta y cuál será el servicio principal que realizará estas acciones, lo que ayuda a configurar automáticamente algunos permisos estándar para ese servicio. Aquí algunos aspectos clave:
>
>Preconfiguración de permisos: Al seleccionar un caso de uso como EC2, IAM aplicará una política inicial que otorga permisos básicos para que EC2 ejecute acciones en tu aplicación (por ejemplo, acceder a registros en S3 o administrar instancias de EC2).
>
>Control de acceso adecuado: Seleccionar un caso de uso ayuda a restringir los permisos del rol exclusivamente al servicio o caso específico que necesites, aplicando el principio de mínimo privilegio, es decir, otorgar solo los permisos necesarios.
>
>Compatibilidad con los servicios: Al especificar un servicio, te aseguras de que los permisos están correctamente alineados con las necesidades de ese servicio, evitando posibles errores de permisos cuando Elastic Beanstalk intenta acceder a otros servicios en AWS.
>
>En resumen, el caso de uso define el contexto en el que el rol será utilizado, asegurando que los permisos estén configurados para que el servicio pueda operar correctamente en tu infraestructura de AWS.
>



> [!TIP]
> Aunque pueda parecer que EC2 y Elastic Beanstalk son cosas separadas, en realidad, Elastic Beanstalk necesita permisos específicos para EC2 porque Beanstalk utiliza instancias de EC2 para ejecutar tu aplicación. Seleccionar EC2 como caso de uso no es inadecuado; en realidad, es necesario y adecuado para darle a Beanstalk los permisos que necesita para administrar esas instancias de EC2.
>
>Elastic Beanstalk es un servicio de orquestación que gestiona varios recursos en AWS, incluyendo:
>
>Instancias de EC2: Para ejecutar el entorno de tu aplicación.
Almacenamiento en S3: Para guardar archivos de configuración, logs, y versiones de la aplicación.
>Otros servicios opcionales: Como RDS para bases de datos o IAM para permisos.
Cuando seleccionas EC2 como caso de uso para el rol IAM, estás indicando que este rol será utilizado principalmente para gestionar las instancias de EC2 asociadas con la aplicación que corre en Elastic Beanstalk. AWS lo interpreta de esa manera y te preconfigura permisos que le permiten a Elastic Beanstalk controlar EC2 y otros servicios necesarios para la infraestructura.
>
>Entonces, seleccionar EC2 como caso de uso para el rol IAM de Elastic Beanstalk no significa que estés dando permisos inapropiados, sino que estás otorgando los permisos mínimos necesarios para que Beanstalk pueda gestionar las instancias EC2 y otros recursos en tu entorno de aplicación.
>



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
