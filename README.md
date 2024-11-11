# Proyecto AWS CI/CD


# 1.0 Beanstalk

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
- Elija un caso de uso para el servicio especificado: `EC2`.

Agregar permisos:
- AdministratorAccess-AWSElasticBeanstalk
- AWSElasticBeanstalkCustomPlatformforEC2Role
- AWSElasticBeanstalkRoleSNS
- AWSElasticBeanstalkWebTier

Detalles del rol:
- Nombre del rol: `gamma-beanstalk-ROL`.
- Descripción: `gamma-beanstalk-ROL`.
  
> [!TIP]
> Al crear un rol IAM para Elastic Beanstalk, debes seleccionar EC2 como caso de uso porque Elastic Beanstalk normalmente administra instancias de EC2 para ejecutar tu aplicación. Esto garantiza que el rol tendrá los permisos correctos para realizar tareas como iniciar, detener y administrar esas instancias EC2 según sea necesario.
>

#### Entonces ¿Para qué sirve el servicio o caso de uso?

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

#### Entonces, si cuando quiero crear un rol para beanstalk ¿Qué sentido tiene seleccionar caso de uso EC2, a caso, no le estaría dando como permisos inadecuados?

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

#### ¿Porqué existe un caso de uso Beanstalk?

>[!TIP]
> El caso de uso "Elastic Beanstalk" en IAM está diseñado para roles específicos que Elastic Beanstalk necesita para gestionar sus propios recursos, pero no para ejecutar las aplicaciones en sí. Este rol de "Elastic Beanstalk" se usa en dos escenarios diferentes:
>- Rol de Servicio de Elastic Beanstalk: Este rol es necesario para que el propio servicio de Elastic Beanstalk tenga permisos para crear y administrar recursos en tu cuenta de AWS. Cuando seleccionas el caso de uso "Elastic Beanstalk" para el rol IAM, estás creando un rol que le permite al servicio Beanstalk gestionar instancias de EC2, asignar direcciones IP, crear balanceadores de carga y otros recursos necesarios para tu entorno.
>
>- Rol de Entorno de Elastic Beanstalk: Este rol se asocia directamente a las instancias EC2 dentro del entorno de tu aplicación y permite que las instancias accedan a otros servicios en AWS según lo necesite la aplicación. Aquí es donde entra el caso de uso "EC2" como opción para el rol IAM, ya que este rol tiene permisos adecuados para que las instancias EC2 (y la aplicación que corren) puedan interactuar con otros recursos, como S3 o DynamoDB.
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

- Service Role: `Use existing service role`, `gamma-beanstalk-ROL`
- EC2 key pair: `clavebeanstalk.pem`.
- EC2 instance profile: `gamma-beanstalk-ROL`.

### Paso 3. Configuración de redes, bases de datos y etiquetas

- VPC: `La predeterminada`.
- Dirección IP pública, `Activado`.
- Seleccionamos todas las "instance subnets".
- Tag: `Project` y `gamma`.
  
### Paso 4. Configuración del escalado y del tráfico de instancias

- Root Volume Type: `General Purpose 3 (SSD)`
- Tipo de entorno: `Equilibrio de Carga`.
- Tipo de instancia: `t2.micro`.
- Visibility Public.

En procesos, vamos a darle al "default" > actions y edit.
(abrimos/desplegamos, el sesiones).
- Session Stickiness: `enabled`
  
### Paso 5. Configuración de actualizaciones, monitoreo y registros

Actualizaciones e implementaciones continuas.
Implementaciones de aplicaciones.
- Política de implementación: `Continuo/Rolling`.
- Tamaño del lote de implementación: `50%`.

# 2.0 RDS

Nos vamos a Amazon RDS, y creamos la base de datos:

Elegir un método de creación de base de datos:
- Creación Estandar.

Opciones del motor:
- MySQL
- MySQL engine version: `8.0.35`
  
Plantilla:
- `Capa gratuita`.

Configuración:
- Identificador de instancias de base de datos: `gamma-rds`.
- Administración de credenciales: `Autoadministrado`.
- Generar Contraseña Automáticamente (HABILITADO).

Configuración de la instancia:
- db.t3.micro

Conectividad:
- Acceso público: `no`.
- Grupo de seguridad de VPC: `Crear nuevo`
- Nuevo nombre del grupo de seguridad de VPC: `gamma-rds-sg`
- DESPLEGAMOS, configuración adicional y comprobamos que sea el puerto: `3306`

Autenticación de bases de datos:
- Autenticación con contraseña.

Configuración adicional:
Opciones de base de datos.
- Nombre de base de datos inicial: `accounts`

Una vez hecho eso, creamos la BD:
Cerraremos el PopUp que nos aparece de "complementos sugeridos para gamma-rds".

Nos aparecerá esto:
`Creación de base de datos gamma-rds
Es posible que el lanzamiento de la base de datos tarde unos minutos.La única forma de ver la contraseña maestra es elegir Ver detalles de credenciales durante la creación de la base de datos. Puede modificar la instancia de base de datos para crear una contraseña nueva en cualquier momento.Puede utilizar la configuración de gamma-rds para simplificar la configuración de complementos de base de datos sugeridos mientras terminamos de crear su base de datos.
Ver detalles de credenciales`

y pues vemos las credenciales y aquí tengo la contraseña:

```
9T1jtPgEgma0kBmeVQIq
```
