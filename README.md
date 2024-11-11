# 0.0 Introducción al proyecto AWS CI/CD. ("gamma").
## 0.1 Explicación del proyecto.
En este proyecto, vamos a usar servicios de AWS para obtener nuestro código, compilarlo y desplegarlo en AWS Elastic Beanstalk. Anteriormente, hemos visto proyectos de CI/CD con Jenkins o pipelines de Jenkins CI/CD.

De la misma manera, vamos a crear una canalización de código (code pipeline), pero vamos a usar servicios de AWS para esto.
Para completar este proyecto:
- Primero vamos a crear un entorno en **AWS Elastic Beanstalk**. Construiremos el código y lo desplegaremos en el entorno de Beanstalk.
- Después, crearemos **AWS RDS** porque nuestra aplicación de perfil necesita conectividad con la base de datos.
- Para el código fuente, utilizaremos repositorios de **Bitbucket**. (que antes estaba en GitHub).
- Luego, vamos a usar el servicio **AWS CodeBuild**, que obtendrá el código fuente de Bitbucket y lo desplegará en Beanstalk.
- Por supuesto, después de construir el artefacto, usaremos el servicio AWS **CodePipeline** para conectar todos estos servicios.
- Y finalmente, vamos a probar nuestra canalización de CI/CD haciendo un commit de git y enviando los cambios a nuestro repositorio de Bitbucket, lo que activará nuestra canalización.

>[!TIP]
>Bitbucket es de Atlassian y es un nombre muy popular entre desarrolladores y también entre ingenieros de DevOps.
>
>Puedes pensar en Bitbucket como GitHub, ya que ofrece soluciones para git. Muchas organizaciones utilizan servicios de Atlassian, y junto con ello, usan el repositorio de Bitbucket.
>
*Por lo tanto, existe una gran probabilidad de que, cuando trabajes en tiempo real, utilices repositorios de Bitbucket o repositorios git en Bitbucket. Además, junto con los repositorios git, Bitbucket proporciona una canalización de CI/CD completa, con varios servicios en esa canalización.*

Antes de comenzar, echemos un vistazo al diagrama arquitectónico.

## 0.2 Arquitectura del proyecto:

Nuestra canalización será así:

Tendremos repositorios git en Bitbucket que migraremos de GitHub a Bitbucket.

El servicio AWS CodeBuild funcionará como Jenkins, obteniendo el código fuente desde los repositorios git de Bitbucket, construyendo nuestro código en un artefacto y luego desplegándolo en Beanstalk.

No hay una opción predeterminada para desplegarlo en Beanstalk.

Usaremos el servicio AWS CodePipeline para conectar todas estas piezas, de modo que CodePipeline detectará cualquier nuevo commit en nuestro repositorio de Bitbucket.

Obtendrá el código fuente, activará el proyecto de CodeBuild, que construirá el código en un artefacto y luego lo desplegará en AWS Elastic Beanstalk.

Como ya sabemos, nuestra aplicación de perfil necesita conectividad de base de datos.

Así que tendremos Amazon RDS, que contendrá esquemas y tablas para nuestra aplicación de perfil web.

# 1.0 Beanstalk.

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

Lo voy a crear desde aquí:

![image](https://github.com/user-attachments/assets/d5507d3d-db9c-4550-bcae-46fae52029ef)

### Paso 1. Configuración del entorno

**Nivel de entorno.**
- Escogemos: `Entorno de servidor web`.

**Información de la aplicación.**
- Application name: `gamma`.

**Información del entorno.**
- Enviroment name: `Gamma-prod11`.
- Domain: `gamma-prod11` y CHECK AVAILABILITY.

**Plataforma.**
- Tipo de plataforma: `Plataforma administrada`.
- Plataforma: `TomCat` (la platform branch Tomcat 10 correto 21 y platform version 5.4.0)

**Código de aplicación.**
- `Sample application` o `Aplicación de ejemplo`.

**Valores preestablecidos.**
- Configuración Personalizada.

### Paso 2. Configuración del acceso al servicio

**Acceso al Servicio**
- Service Role: `Use existing service role`, `gamma-beanstalk-ROL` (*El que creamos en el apartado 1.1*).
- EC2 key pair: `clavebeanstalk.pem`.
- EC2 instance profile: `gamma-beanstalk-ROL`.

### Paso 3. Configuración de redes, bases de datos y etiquetas

**Nube virtual privada (VPC)**
- VPC: `La predeterminada`.

**Configuración de la instancia.**
- Dirección IP pública, `ACTIVADO`.
- Seleccionamos todas las "instance subnets".

**NOS SALTAMOS "BASE DE DATOS"**

**Etiquetas.**
- Clave: `Project` , Valor: `gamma`.
  
### Paso 4. Configuración del escalado y del tráfico de instancias

**Instancias.**
- Root Volume Type: `General Purpose 3 (SSD)`

**Capacidad.**
- Tipo de entorno: `Equilibrio de Carga`. `Min 2, Max 4.`
- Tipo de instancia: `t2.micro`.
- Visibility Public.

**Tipo de equilibrador de carga.**
- `Equilibrador de carga de aplicación y Dedicado`.

**Procesos.**
En procesos, **vamos a darle al "default"**, luego `actions` y `edit`.
(abrimos/desplegamos, el sesiones).
- Session Stickiness: `enabled`
  
### Paso 5. Configuración de actualizaciones, monitoreo y registros

`Monitorización`, `Actualizacones adminsitradas de la plataforma`, `Notificaciones por correo electrónico.`**Nos lo saltamos**.

**Actualizaciones e implementaciones continuas.**
Implementaciones de aplicaciones.
- Política de implementación: `Continuo/Rolling`.
- Tamaño del lote de implementación: `50%`.

TODO LO DEMÁS LO SALTAMOS.

# 2.0 RDS
## 2.1 Crear RDS.
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

>Creación de base de datos gamma-rds
>Es posible que el lanzamiento de la base de datos tarde unos minutos.La única forma de ver la contraseña maestra es elegir Ver detalles de credenciales durante la creación de la base de datos. Puede modificar la instancia de base de datos para crear una contraseña nueva en cualquier momento.Puede utilizar la configuración de gamma-rds para simplificar la configuración de complementos de base de datos sugeridos mientras terminamos de crear su base de datos.
>
>Ver detalles de credenciales

y pues vemos las credenciales y aquí tengo la contraseña:

```
9T1jtPgEgma0kBmeVQIq
```

## 2.2 Inicializar la base de datos.

Ahora vamos a tener que modificar el Security Group, nos vamos a EC2, luego a Security Groups, buscamos el grupo de seguridad, CUYA DESCRIPCIÓN SEA `VPC Security Group`.

copiamos su ID de Grupo de Seguridad:
```
sg-099e3f53d04486ee7
```

Ahora nos vamos a la que acabamos de crear, la RDS, `gamma-rds-sg`. Editamos las reglas de entrada. Y añadimos una:

- Type: `Custom TCP`.
- Port Range: `3306`.
- Source: `Custom` , `sg-099e3f53d04486ee7`.

Guardamos la regla.

Esto lo hacemos para que nuestro Beanstalk pueda conectarse a nuestro RDS.

Volvemos a las instancias y ahora selecionamos 1, y vamos a conectarnos desde allí para así poder inicializar la BD. Accederemos por SSH.

```
ssh -i "clavebeanstalk.pem" ec2-user@44.192.246.161
```

Tiene que ser obligatoriamente con el usuario "ec2-user", no te deja con root, por lo tanto no me sirve copiar y pegar el comando del AWS, hay que modificarlo un poco.

```
sudo -i
```

```
dnf search mysql
```

```
dnf install mariadb105 -y
```

Nos vamos a Amazon RDS, y buscamos el punto de enlace de la Base de datos:

```
mysql -h gamma-rds.crmqiuq428z2.us-east-1.rds.amazonaws.com -u admin -p9T1jtPgEgma0kBmeVQIq accounts
```

voy a hacer un `SHOW databases;`

este es el output:
```
MySQL [accounts]> SHOW databases;
+--------------------+
| Database           |
+--------------------+
| accounts           |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
Nos salimos con un `exit`, hemos podido ver la base de datos "accounts".
Ahora voy a GitHub, que es donde está el .sql:

![image](https://github.com/user-attachments/assets/ee045c22-b7ab-4c6a-bac0-c04c811ebde0)

Voy a darle a Raw y nos abre así:

![image](https://github.com/user-attachments/assets/fd8b579c-6fca-4bd3-9a48-14ba6864707b)

y copio ese enlace:

```
https://raw.githubusercontent.com/hkhcoder/vprofile-project/refs/heads/aws-ci/src/main/resources/db_backup.sql
```

y ahora hago un:
```
wget https://raw.githubusercontent.com/hkhcoder/vprofile-project/refs/heads/aws-ci/src/main/resources/db_backup.sql
```

y pues voy a ponerlo en accounts:
```
mysql -h gamma-rds.crmqiuq428z2.us-east-1.rds.amazonaws.com -u admin -p9T1jtPgEgma0kBmeVQIq accounts < db_backup.sql
```
y este es el output, porque he vuelto a iniciar sesión para comprobar que todo se ha hecho correctamente:

```
MySQL [accounts]> USE accounts;
Database changed

MySQL [accounts]> SHOW TABLES;
+--------------------+
| Tables_in_accounts |
+--------------------+
| role               |
| user               |
| user_role          |
+--------------------+
3 rows in set (0.002 sec)
```

# 3.0 BitBucket.

Vamos a tener que crear un Workspace:

![image](https://github.com/user-attachments/assets/9772261a-f9df-4022-ad97-cfa3cddbacf9)

Voy a crear un repositorio, dentro de ese WorkSpace:

![image](https://github.com/user-attachments/assets/717581df-695f-4f1a-9688-92a32b2e333a)

Necesitamos que el respositorio esté completamente vacío, porque queremos simplemente subir nuestro proyecto a BitBucket.

![image](https://github.com/user-attachments/assets/744e47a2-1c63-4411-9066-b4e777f833db)

## 3.1 Generar pares-clave ssh.

este es el output y comando:

```
Usuario@DESKTOP-HFA11LU MINGW64 ~/Desktop$ ssh-keygen

Generating public/private ed***** key pair.
Enter file in which to save the key (/c/Users/Usuario/.ssh/id_ed*****): gamma-rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in gamma-rsa
Your public key has been saved in gamma-rsa.pub
The key fingerprint is:
SHA256:L5U             lo censuro por si a caso.    Usuario@DESKTOP-******
The key's randomart image is:
+--[ED***** 256]--+
|  .o+=B..        |
|   .o+o= =       |
|     AA A A A    |
|     A a A A     |
|      A A a A    |
|     A A a A A   |
|      . * E .    |
|     . + o .     |
|      =o.        |
+----[SHA256]-----+
```
