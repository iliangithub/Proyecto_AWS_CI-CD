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

> [!IMPORTANT]
> Tienes que ejecutar el comando de keygen y poner el config en esta ruta:
>```
>cd /c/Users/Usuario/.ssh
>```


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

voy a hacer un cat gamma-rsa.pub, para ver la clave pública.

```
Usuario@DESKTOP-HFA11LU MINGW64 ~/Desktop
$ cat gamma-rsa.pub
ssh-ed25519 **************************************IfRAqn2cD7NPajlDytbe+Mi8y3bkbutVfKn Usuario@DESKTOP-*****
```

Ahora, voy a volver a BitBucket:

![image](https://github.com/user-attachments/assets/5de4d046-593d-4048-88f7-9f4964c4b4d2)

Luego, SSH keys, añado:

![image](https://github.com/user-attachments/assets/45a0cc8e-78bd-4f37-a0a0-145eb63319e2)

y ahora, necesitamos hacer un "configfile", entonces, cuando hagamos "git push" o "git pull", en los repositorios de bitbucket , pues necesitamos usar las claves privadas que acabamos de crear antes, para logearnos.

![image](https://github.com/user-attachments/assets/6116c76a-d74b-4edb-8e21-7bfb04de6884)

Ponemos esto:

```
#bitbucket.org
Host bitbucket.org
 PreferredAuthentications publickey
 IdentityFile ~/.ssh/gamma-rsa
```

Ahora, nos volvemos al repositorio y vamos a pues introducir.

![image](https://github.com/user-attachments/assets/5d4b1319-f7a0-4bd7-8944-48bae360bb8f)

copio el comando, y usamos SSH.

```
ssh -T git@bitbucket.org
```

> [!WARNING]
> ### Posible error:
> Si nos sale este error:
> ```
> $ ssh -T git@bitbucket.org
>The authenticity of host 'bitbucket.org (185.166.143.50)' can't be established.
>ED25519 key fingerprint is SHA256:ybgmFkzwOSotHTHLJgHO0QN8L0xErw6vd0VhFA9m3SM.
>This key is not known by any other names.
>Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
>Warning: Permanently added 'bitbucket.org' (ED25519) to the list of known hosts.
>git@bitbucket.org: Permission denied (publickey).
> ```
> **ES PORQUE NO LO HEMOS HECHO EN EL DIRECTORIO .ssh**

al ejecutar `ssh -T git@bitbucket.org`

nos tiene que salir:

```
$ ssh -T git@bitbucket.org
authenticated via ssh key.

You can use git to connect to Bitbucket. Shell access is disabled
```

Y ahora NOS MOVEMOS A OTRO DIRECTORIO... /tmp/

```
cd /tmp
```

```
git clone git@bitbucket.org:aws_devops_ci-cd/gamma-aplicacion.git
```

![image](https://github.com/user-attachments/assets/0aa61963-d9bb-4201-ba1e-df360c365a65)

Y ahora si hago un cat:

```
$ cat gamma-aplicacion/.git/config
[core]
        repositoryformatversion = 0
        filemode = false
        bare = false
        logallrefupdates = true
        symlinks = false
        ignorecase = true
[remote "origin"]
        url = git@bitbucket.org:aws_devops_ci-cd/gamma-aplicacion.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
```

![image](https://github.com/user-attachments/assets/c780051d-e160-46bd-9f1a-7bd22f478070)

![image](https://github.com/user-attachments/assets/3b1a35c6-4783-4663-b1d2-f9f533c841a0)

Ahora, vamos a cambiar a la rama `aws-ci`

```
git checkout aws-ci
```

Bueno si hacemos un:

```
git branch -a
```
Podemos ver todas las ramas.

Si nuestro repositorio tiene "tags", podemos hacer un `git fetch --tags` (lo ejecutamos).

```
git remove rm origin
```

Y ahora la URL, está borrada:

```
$ cat .git/config
[core]
        repositoryformatversion = 0
        filemode = false
        bare = false
        logallrefupdates = true
        symlinks = false
        ignorecase = true
```

```
git remote add origin git@bitbucket.org:aws_devops_ci-cd/gamma-aplicacion.git
```

y si volvemos a hacer un cat:
```
$ cat .git/config
[core]
        repositoryformatversion = 0
        filemode = false
        bare = false
        logallrefupdates = true
        symlinks = false
        ignorecase = true
[remote "origin"]
        url = git@bitbucket.org:aws_devops_ci-cd/gamma-aplicacion.git
        fetch = +refs/heads/*:refs/remotes/origin/*
```
aparece la URL.

```
$ git push origin --all
Enumerating objects: 787, done.
Counting objects: 100% (787/787), done.
Delta compression using up to 20 threads
Compressing objects: 100% (448/448), done.
Writing objects: 100% (787/787), 11.60 MiB | 9.62 MiB/s, done.
Total 787 (delta 233), reused 787 (delta 233), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (233/233), done.
To bitbucket.org:aws_devops_ci-cd/gamma-aplicacion.git
 * [new branch]      aws-ci -> aws-ci
 * [new branch]      main -> main
```

Si nos vamos al BitBucket:

![image](https://github.com/user-attachments/assets/d8869926-c374-4a8b-90c4-2ec57e0724f1)

# 4.0 AWS CodeBuild.

Vamos a usar el servicio AWS CodeBuild para obtener nuestro código fuente y construirlo en un artefacto.

Entonces, busca CodeBuild y ábrelo.

No necesito explicarte CodeBuild desde cero porque ya conoces Jenkins. Es la versión en la nube de AWS de Jenkins.

La diferencia, o la principal diferencia, diría yo, es Jenkins. En Jenkins necesitamos lanzar una instancia o necesitamos una máquina virtual o un ordenador para ejecutar el servicio de Jenkins. CodeBuild es diferente.

Es un servicio de pago por uso. Creas un proyecto de CodeBuild, que es como un trabajo en Jenkins. Y cuando ejecutas ese proyecto o ejecutas la compilación, el recurso de cómputo que usa, la RAM, la memoria, la CPU, lo que sea que utilice, solo pagas por eso durante el tiempo que se use.

Mientras tu compilación esté en ejecución, solo pagas por el tiempo de compilación, así que no necesitas tener continuamente una instancia de EC2 en ejecución.

Si ves aquí, dice que AWS CodeBuild es un servicio de integración continua totalmente gestionado que compila el código fuente, ejecuta pruebas, produce paquetes de software y dice que pagas solo por el tiempo de compilación que usas.

Y es bastante simple.

Lo principal en CodeBuild es el archivo buildspec.

## 4.1 Crear S3 Bucket.

Nuestro artefacto va a estar en S3 bucket. Creamos un bucket:

**Configuración General.**
- Región de AWS: EE. UU. Este (Norte de Virginia) us-east-1
>[!IMPORTANT]
> Es importante que el S3 bucket esté creado en la misma región que el "CodeBuild"
- Tipo de bucket: `Uso general`
- Nombre del bucket: `gamma-aws-ci-cd-artifact`

Y no hay nada más que hacer.

## 4.2 Crear el CodeBuild.

![image](https://github.com/user-attachments/assets/6b489c94-d2a8-4abb-8e96-593908c30eef)

**Configuración del proyecto**
- Nombre del proyecto: `gammacicd-build`
- 
**Origen**
Aquí es curioso, porque desde el tutorial, cambia mucho la forma de hacer las cosas:
![image](https://github.com/user-attachments/assets/b4115fbc-abe2-4d0e-a43c-47498a539067)

![image](https://github.com/user-attachments/assets/cab2609d-dffe-4081-a503-9245313df8d4)

![image](https://github.com/user-attachments/assets/a63e77c3-c16e-49cd-8a0d-57e7a5b8bf63)

Y el nombre de la conexión: `conexion-gamma-bitbucket-ci-cd` y voy a darle pues a conectarse:

![image](https://github.com/user-attachments/assets/a1ed8b6e-f754-411b-94a2-d300d17e3aef)

Grant access.

![image](https://github.com/user-attachments/assets/cc8533e3-7bf2-40dd-aeb6-166a1cebc49a)

evidentemente, seleccionamos esa workspace.

![image](https://github.com/user-attachments/assets/ef25812c-5fc8-4194-a395-6bad5d1fdbc2)

y `grant access`

![image](https://github.com/user-attachments/assets/05947a98-227f-46f3-9129-2b0c6bcb7f54)

y conectarse.

Tras esto, finalmente tenemos la conexión hecha:

![image](https://github.com/user-attachments/assets/043bc3fe-e5a3-418a-9517-85203807ecfc)

![image](https://github.com/user-attachments/assets/ea3f4464-ba53-4504-a924-5da20c574486)

![image](https://github.com/user-attachments/assets/8392040b-bd7f-4926-a00e-c555904f1703)

En la parte de "Especificación de compilación".

![image](https://github.com/user-attachments/assets/d167e4ec-e672-4c3a-a66d-8d73ba962fa9)

Le damos al editor.

aquí tenemos por ejemplo la documentación por si acaso:

https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html

necesitamos el endpoint del RDS:

```
gamma-rds.crmqiuq428z2.us-east-1.rds.amazonaws.com
```

```
version: 0.2

#env:
  #variables:
     # key: "value"
     # key: "value"
  #parameter-store:
     # key: "value"
     # key: "value"

phases:
  install:
   runtime-versions:
      java: corretto17
  pre_build:
    commands:
      - apt-get update
      - apt-get install -y jq 
      - wget https://archive.apache.org/dist/maven/maven-3/3.9.8/binaries/apache-maven-3.9.8-bin.tar.gz
      - tar xzf apache-maven-3.9.8-bin.tar.gz
      - ln -s apache-maven-3.9.8 maven
      - sed -i 's/jdbc.password=admin123/jdbc.password=contraseñadelRDSquehemoscreadoantessssss/' src/main/resources/application.properties
      - sed -i 's/jdbc.username=admin/jdbc.username=admin/' src/main/resources/application.properties
      - sed -i 's/db01:3306/vprodb.c50sgqqusvnr.us-east-1.rds.amazonaws.com:3306/' src/main/resources/application.properties
  build:
    commands:
      - mvn install
  post_build:
    commands:
       - mvn package
artifacts:
  files:
     - '**/*'
  base-directory: 'target/vprofile-v2'
```

y este es en mi caso como quedaría el código:

```
version: 0.2

#env:
  #variables:
     # key: "value"
     # key: "value"
  #parameter-store:
     # key: "value"
     # key: "value"

phases:
  install:
   runtime-versions:
      java: corretto17
  pre_build:
    commands:
      - apt-get update
      - apt-get install -y jq 
      - wget https://archive.apache.org/dist/maven/maven-3/3.9.8/binaries/apache-maven-3.9.8-bin.tar.gz
      - tar xzf apache-maven-3.9.8-bin.tar.gz
      - ln -s apache-maven-3.9.8 maven
      - sed -i 's/jdbc.password=admin123/jdbc.password=9T1jtPgEgma0kBmeVQIq/' src/main/resources/application.properties
      - sed -i 's/jdbc.username=admin/jdbc.username=admin/' src/main/resources/application.properties
      - sed -i 's/db01:3306/gamma-rds.crmqiuq428z2.us-east-1.rds.amazonaws.com:3306/' src/main/resources/application.properties
  build:
    commands:
      - mvn install
  post_build:
    commands:
       - mvn package
artifacts:
  files:
     - '**/*'
  base-directory: 'target/vprofile-v2'
```
** **
** **
** **
** **
** **
** **
** **
** **
