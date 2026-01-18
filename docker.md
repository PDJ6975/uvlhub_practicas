Tutorial-docker
Contenido
1	Ejercicio 1: Empezamos desde cero
1.1	Eliminar la configuración existente
1.2	Instalar Docker y Docker Compose (Linux)
1.3	Crear un Dockerfile mínimo
1.3.1	FROM
1.3.2	WORKDIR
1.3.3	COPY
1.3.4	RUN
1.3.5	EXPOSE
1.3.6	CMD
1.3.7	RUN vs CMD
1.3.7.1	RUN
1.3.7.2	CMD
1.3.7.3	Diferencia resumida
1.4	Dockerfile completo
1.5	Construir la imagen
1.6	Ejecutar el contenedor
2	Ejercicio 2: Conectando servicios
2.1	Crear contenedor MariaDB
2.2	Descargar la imagen oficial
2.3	Crear el contenedor de MariaDB
2.4	Comprobar que está corriendo
2.5	Conectarse a la base de datos
2.6	Verificar la base de datos
2.7	Redes internas
2.7.1	Conectar servicios
3	Ejercicio 3: Persistencia
3.1	Volúmenes
3.1.1	Crear un volumen
3.1.2	Crear un contenedor de MariaDB con el volumen
3.2	Qué significa ":" en los volúmenes =
4	Ejercicio 4: Orquestación de servicios
4.1	Orquestación básica
4.2	Dependencias entre contenedores y entrypoints
4.3	Trabajando con CMD, entrypoints y commands
4.3.1	CMD → “qué hacer por defecto”
4.3.2	ENTRYPOINT → “qué se ejecuta siempre”
4.3.3	command (de docker-compose.yml) → “sobrescribe el CMD”
4.4	Combinación de elementos
5	Ejercicio 5: Volumen de trabajo
5.1	Problema de desarrollo
5.2	Crear volumen de trabajo (bind mount)
5.3	Cuidado con COPY y los bind mounts
6	Ejercicio 6: Distintas configuraciones de despliegue
6.1	Autoevaluación
6.2	Ejercicio propuesto
6.3	Comentarios finales
6.3.1	Cuanto más se parezca el entorno de desarrollo al de producción, menos errores habrá
6.3.2	Diferencia entre el servidor de Flask y Gunicorn
Ejercicio 1: Empezamos desde cero
El objetivo es eliminar toda la configuración previa y crear desde cero la estructura de carpetas que contendrá nuestros futuros archivos Docker.

Eliminar la configuración existente
Antes de comenzar, asegúrate de borrar cualquier carpeta Docker existente en el proyecto. En este caso, le haremos una copia de seguridad para poder volver a ella.

mv docker docker.bk
También debes asegurarte que trabajas con las variables de entorno adecuadas:

cp .env.docker.example .env
⚠️ Este paso es importante: partimos completamente desde cero para entender cada parte del sistema.

Instalar Docker y Docker Compose (Linux)
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce
sudo usermod -aG docker ${USER}
mkdir -p ~/.docker/cli-plugins/
LATEST_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
curl -SL "https://github.com/docker/compose/releases/download/${LATEST_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
Es posible que tengas que reiniciar la sesión de usuario para poder usar docker sin necesidad de sudo.

Crear un Dockerfile mínimo
Ahora crea nuevamente la carpeta base donde almacenaremos los archivos relacionados con Docker:

 mkdir -p docker/images
Creamos un Dockerfile vacío

touch docker/images/Dockerfile
Vamos a explicar los distintos apartados:

FROM
FROM python:3.12-slim
Indica la imagen base sobre la que se construirá la tuya. En este caso, parte de una imagen ligera que ya tiene Python 3.12 instalado.

WORKDIR
WORKDIR /app
Esta línea define el directorio de trabajo dentro del contenedor. A partir de aquí, todos los comandos que aparezcan después (COPY, RUN, CMD, etc.) se ejecutarán dentro de /app.

Ojo: /app no es la carpeta app/ de tu proyecto local.. Son cosas distintas.

En tu máquina puede existir algo como:

uvlhub
   app
   core
   requirements.txt
Pero dentro del contenedor la estructura será distinta:

/ 
  app/ 
    app/ ← aquí dentro se habrá copiado tu carpeta local "app"
Cuando más adelante uses:

COPY . .
Docker copiará todos los archivos de tu proyecto dentro de la carpeta /app del contenedor. Por eso el código quedará en /app/app/.

COPY
COPY . .
Copia todos los archivos del proyecto (del host) dentro del contenedor, en /app. El primer punto es el origen; el segundo, el destino.

RUN
RUN pip install --no-cache-dir -r requirements.txt
Ejecuta comandos durante la construcción de la imagen. Aquí instalamos las dependencias necesarias desde requirements.txt. El resultado de este paso quedará guardado en la imagen.

EXPOSE
EXPOSE 5000
Documenta el puerto que la aplicación usa dentro del contenedor. Esto no lo abre al exterior; simplemente indica que la app escucha en el 5000.

CMD
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000", "--reload", "--debug"]
Define el comando que se ejecutará cuando el contenedor se inicie. En este caso, ejecutará la aplicación Flask.

RUN vs CMD
Estas dos instrucciones parecen similares, pero hacen cosas muy distintas.

RUN
Se ejecuta durante la construcción de la imagen (en el momento del docker build). Cada RUN crea una nueva capa dentro de la imagen.

Ejemplo:

RUN pip install -r requirements.txt
Este comando instala las dependencias una sola vez, cuando se construye la imagen. El resultado (las librerías instaladas) queda grabado dentro de la imagen final.

Si después lanzas un contenedor nuevo a partir de esa imagen, las dependencias ya estarán instaladas y no se volverán a ejecutar.

CMD
Se ejecuta cuando se inicia el contenedor (en el docker run). Define el proceso principal que se ejecutará dentro del contenedor mientras esté activo.

Ejemplo:

CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
Este comando no se ejecuta durante la construcción, sino cada vez que arrancas el contenedor. Si detienes el contenedor, este proceso también se detiene.

Diferencia resumida
RUN → se ejecuta al construir la imagen (fase de docker build).

CMD → se ejecuta al iniciar el contenedor (fase de docker run).

Solo puede haber un CMD por Dockerfile. Si hay más de uno, solo se usará el último.

Dockerfile completo
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 5000
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000", "--reload", "--debug"]
Esto tiene que ir en un Dockerfile en docker/images

Construir la imagen
Antes de construir la imagen, asegúrate de que ignoramos el módulo webhook:

echo "webhook" > .moduleignore

Desde la raíz del proyecto (no dentro de docker/images):

docker build -t uvlhub:dev -f docker/images/Dockerfile .
Ejecutar el contenedor
docker run -p 5000:5000 uvlhub:dev
Abre el navegador en http://localhost:5000. ¿Qué es lo que observas?

Ejercicio 2: Conectando servicios
¡Vaya! ¡No funciona! Can't connect to MySQL server on 'db'... ¡Ah, claro, intenta conectarse a la base de datos, que no lo tenemos! Pero claro, en un contenedor no es recomendable tener la app y la base de datos conviviendo...

Crear contenedor MariaDB
Antes que nada, haz Ctrl + C para parar el contenedor web. En este ejercicio aprenderás a lanzar un contenedor de MariaDB 12.0.2 de forma manual, configurando todas las variables necesarias para tu aplicación Flask.

El objetivo es entender la complejidad de hacerlo “a mano”, antes de automatizarlo con docker-compose.

Descargar la imagen oficial
Primero descarga la versión exacta que usaremos:

docker pull mariadb:12.0.2
Crear el contenedor de MariaDB
Lanza un contenedor con todas las variables necesarias para que Flask pueda conectarse correctamente:

docker run -d --name mariadb_container \
 -e FLASK_APP_NAME="UVLHUB.IO(dev)" \
 -e FLASK_ENV=development \
 -e DOMAIN=localhost \
 -e MARIADB_HOSTNAME=db \
 -e MARIADB_PORT=3306 \
 -e MARIADB_DATABASE=uvlhubdb \
 -e MARIADB_TEST_DATABASE=uvlhubdb_test \
 -e MARIADB_USER=uvlhubdb_user \
 -e MARIADB_PASSWORD=uvlhubdb_password \
 -e MARIADB_ROOT_PASSWORD=uvlhubdb_root_password \
 -e WORKING_DIR=/app/ \
 -p 3306:3306 mariadb:12.0.2
Explicación:

-d: ejecuta el contenedor en segundo plano (modo detached).
--name: asigna un nombre identificable al contenedor.
-e: define variables de entorno dentro del contenedor.
-p 3306:3306: publica el puerto de MariaDB en tu máquina local.
mariadb:12.0.2: imagen exacta que vamos a usar.
Aquí ya, de entrada, vemos que esto es más feo que una nevera por detrás. Para empezar, estamos pasando a lo loco un montón de variables que difílmente será replicable en producción, por contener información sensible. Aparte, si metemos nuevas variables, ese comando queda inservible. Necesitaremos una solución más ágil para este problema

Comprobar que está corriendo
docker ps
Deberías ver algo así:

CONTAINER ID IMAGE COMMAND STATUS PORTS NAMES
abcd1234efgh mariadb:12.0.2 "docker-entrypoint.s…" Up 5 seconds 0.0.0.0:3306->3306/tcp mariadb_container
Conectarse a la base de datos
Abre una consola dentro del contenedor:

docker exec -it mariadb_container bash
Y dentro, accede al cliente de MariaDB:

mariadb -u root -p
Contraseña: uvlhubdb_root_password

Verificar la base de datos
Una vez dentro del cliente MySQL, ejecuta:

SHOW DATABASES;
Deberías ver que existen la bases de datos uvlhubdb porque pedimos crearla a la hora de levantar el contenedor. ¿Qué tiene la base de datos dentro? Vamos a verlo...

USE uvlhubdb;
SHOW TABLES;
Un total de... 0 tablas... ¡Claro! Es que solo hemos creado la base en sí, pero no las tablas. ¿Con un script SQL tal vez? ¡Espera, espera, si tenemos las migraciones! Pero claro, para eso necesito el contenedor de la app levantado. Sal de la consola SQL y del contenedor de MariaDB. Bien, ahora en el host...

docker run -p 5000:5000 -d uvlhub:dev
Esta vez con el flag "-d" para que corra en segundo plano. Ahora, accedamos al contenedor y ejecutemos las migraciones. Pero... ¿cómo se llama mi contenedor?

docker ps
Fíjate que Docker asigna un nombre aleatorio a tu contenedor web porque no definiste ninguno. Una vez identificado el nombre:

docker exec -it <nombre> bash
flask db upgrade
What? ¿Qué pasa? ¿Otra vez error de conexión con la base? Si tenemos el contenedor funcionando, esto es un jaleo...

Redes internas
Ahora mismo tienes dos contenedores aislados, el de la app y el la base de datos. Por defecto, cada contenedor tiene su propia red interna y no puede ver a los demás por nombre. El hostname db que usa Flask no existe en su DNS interno.

Conectar servicios
Vamos a crear una red interna que compartan ambos contenedores

docker network create uvlhub_network
docker network ls
Vemos que ya aparece nuestra red uvlhub_network. Ahora, conectemos el contenedor de MariaDB a la red:

docker stop mariadb_container
docker rm mariadb_container
docker run -d \
  --name mariadb_container \
  --hostname db \
  --network uvlhub_network \
  -e FLASK_APP_NAME="UVLHUB.IO(dev)" \
  -e FLASK_ENV=development \
  -e DOMAIN=localhost \
  -e MARIADB_HOSTNAME=db \
  -e MARIADB_PORT=3306 \
  -e MARIADB_DATABASE=uvlhubdb \
  -e MARIADB_TEST_DATABASE=uvlhubdb_test \
  -e MARIADB_USER=uvlhubdb_user \
  -e MARIADB_PASSWORD=uvlhubdb_password \
  -e MARIADB_ROOT_PASSWORD=uvlhubdb_root_password \
  -e WORKING_DIR=/app/ \
  -p 3306:3306 \
  mariadb:12.0.2
Claro, si te fijas, ahora hay que añadir, además de toda esa configuración engorrosa, que se añada en la misma red y que además tenga de hostname "db" para que el contenedor web entienda qué es "db". Tenemos que hacer lo mismo para el contenedor web:

docker stop <nombre>
docker rm <nombre>
Y lo levantamos de nuevo, pero ya en la nueva red. De paso, le ponemos un nombre para identificarlo mejor.

docker run -p 5000:5000 --name web_app_container --network uvlhub_network -d uvlhub:dev
Y ahora ya podemos acceder al contenedor por su nombre y ejecutar las migraciones:

docker exec -it web_app_container bash
flask db upgrade
Abre el navegador en http://localhost:5000. ¿Mejor?

Ejercicio 3: Persistencia
Vamos a simular que hemos desplegado justo esto en producción y el servidor se reinicia por un corte de luz. Reiniciemos a mano cada contenedor:

docker restart web_app_container
docker restart mariadb_container
Abre de nuevo http://localhost:5000. What? ¿Cómo? ¿Qué ha pasado? ¿Otra vez problema de conexión? Ve dándole a F5... F5... ¡bingo! Ahora sí sale, pero menuda aleatoriedad, ¿no? Claro, es que ahora caes en que una cosa es que el contenedor haya arrancado y otra que esté listo.

Aparte, tenemos otro problema. ¿Qué pasa con la persistencia? Porque si borro el contenedor de MariaDB, todos los datos que estén en su interior también se pierden. Y si estamos en producción, lío asegurado.

Volúmenes
Cada vez que eliminas el contenedor de MariaDB, todas las bases de datos se pierden. Esto ocurre porque los datos se guardan dentro del sistema de archivos del contenedor, y al borrarlo desaparecen con él.

En este ejercicio creas un volumen de Docker para que los datos persistan entre ejecuciones.

Crear un volumen
Un volumen es una carpeta gestionada por Docker que vive fuera del ciclo de vida de los contenedores.

Ejecuta:

docker volume create mariadb_data
Comprueba que existe:

docker volume ls
La salida muestra algo como:

DRIVER    VOLUME NAME
local     mariadb_data
Crear un contenedor de MariaDB con el volumen
Lanza el contenedor y conecta el volumen a la ruta donde MariaDB guarda los datos (/var/lib/mysql):

docker stop mariadb_container
docker rm mariadb_container
docker run -d \
  --name mariadb_container \
  --hostname db \
  --network uvlhub_network \
  -v mariadb_data:/var/lib/mysql \
  -e FLASK_APP_NAME="UVLHUB.IO(dev)" \
  -e FLASK_ENV=development \
  -e DOMAIN=localhost \
  -e MARIADB_HOSTNAME=db \
  -e MARIADB_PORT=3306 \
  -e MARIADB_DATABASE=uvlhubdb \
  -e MARIADB_TEST_DATABASE=uvlhubdb_test \
  -e MARIADB_USER=uvlhubdb_user \
  -e MARIADB_PASSWORD=uvlhubdb_password \
  -e MARIADB_ROOT_PASSWORD=uvlhubdb_root_password \
  -e WORKING_DIR=/app/ \
  -p 3306:3306 \
  mariadb:12.0.2
Qué significa ":" en los volúmenes =
Cuando usas la opción -v o --volume en Docker, el signo dos puntos (:) separa dos rutas:

-v origen:destino

origen → es la ruta o el volumen en tu máquina (el host).

destino → es la ruta dentro del contenedor.

Ejercicio 4: Orquestación de servicios
Hasta ahora hemos creado y conectado los contenedores manualmente. Ahora vas a usar Docker Compose para definir todo en un solo archivo.

Lo primero es parar y eliminar los contenedores previos para que no interfieran:

docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
Orquestación básica
Crea un archivo llamado docker-compose.yml en la carpeta docker con este contenido:

services:
  web:
    container_name: web_app_container
    env_file:
      - ../.env
    expose:
      - "5000"
    depends_on:
      - db
    build:
      context: ../
      dockerfile: docker/images/Dockerfile
    networks:
      - uvlhub_network

  db:
    container_name: mariadb_container
    image: mariadb:12.0.2
    env_file:
      - ../.env
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - uvlhub_network

volumes:
  db_data:

networks:
  uvlhub_network:
Fíjate que esta orquestación de servicios me permite definir el nombre de cada servicio, su dependencia, sus puertos, sus volúmenes, redes... todo lo que hacíamos de forma manual y propenso a errores queda ahora más fácil de gestionar. Además, ahora ambos contenedores comparten el mismo archivo .env de variables de entorno.

Bien, ahora vamos a levantar estos contenedores y acceder al contenedor web:

docker compose -f docker/docker-compose.yml up -d --build
docker exec -it web_app_container bash
Inmediatamente después, ejecuta las migraciones:

flask db upgrade
¿Qué ha ocurrido? Depende del tiempo que hayas tardado, puede que haya saltado un error o puede que haya funcionado. Puede que hayas intentado varias veces el comando anterior hasta que ha funcionado. ¿Por qué crees que ocurre esta aparente aleatoriedad?

Dependencias entre contenedores y entrypoints
Fíjate que tenemos un "depends on" en el servicio web. Significa que el servicio "db" debe iniciarse antes que el servicio web. Pero, cuidado, ahí está la trampa, porque "servicio iniciado != servicio preparado". El servicio de MariaDB tiene que arrancar, configurarse, etc...

Cuando hicimos el comando COPY en nuestro Dockerfile, también copiamos la carpeta scripts. Uno de los scripts es wait-for-db.sh, que realiza continuamente llamadas a MariaDB hasta que este servicio está listo para recibir peticiones. Ábrelo y estúdialo.

Añade al servicio web de tu docker compose las siguientes líneas:

entrypoint: ["/app/scripts/wait-for-db.sh"]
command: ["flask", "run", "--host=0.0.0.0", "--port=5000", "--reload", "--debug"]
A su vez, del Dockerfile, hay que eliminar la línea de CMD que arrancaba nuestro servidor Flask.

Luego, reinicializa todos los servicios:

docker compose -f docker/docker-compose.yml down
docker compose -f docker/docker-compose.yml up -d --build
Abre de nuevo http://localhost:5000. What? ¿No abre? Vamos a investigarlo, veamos el logs:

docker logs web_app_container
Nos debe salir un mensaje eterno de "/app/scripts/wait-for-db.sh: line 18: mariadb: command not found". Esto ocurre porque en la imagen de nuestro servicio web NO TENEMOS el cliente de MariaDB. Debemos indicar entonces que lo instale. Añade justo debajo de FROM lo siguiente:

RUN apt-get update && apt-get install -y --no-install-recommends mariadb-client
Al ser un contenedor basado en Linux, y accediendo como root, es como si actualizáramos todos los paquetes y luego instalásemos el cliente de MariaDB. Intentémoslo otra vez:

docker compose -f docker/docker-compose.yml down
docker compose -f docker/docker-compose.yml up -d --build
Abre de nuevo http://localhost:5000. ¡Jopelines! ¿Tampoco? Si haces un "docker ps" verás el problema. El contenedor web no tiene expuesto ningún puerto.

services:
  web:
    (...)
    expose:
      - "5000"
    (...)
"Expose" hace visible el puerto entre contenedores, pero NO entre tu contenedor y el host. Para eso usamos la directiva "ports", que se encarga del mapping. En nuestro caso, será:

services:
  web:
    (...)
    ports:
      - "5000:5000"
    (...)
En ocasiones, si los dos puertos coinciden, se puede dejar uno solo con "5000" pero dejaremos los dos. El de la izquierda siempre es el puerto externo (host), el de la derecha es el puerto interno (contenedor). Así que cambia ese detalle y por última vez:

docker compose -f docker/docker-compose.yml down
docker compose -f docker/docker-compose.yml up -d --build
Recuerda ejecutar las migraciones:

docker exec -it web_app_container bash
flask db upgrade
exit
Ten en cuenta que si falla, es porque la base aún no está lista, para eso tenemos el script... Y, ya que estamos, ¿se te ocurre alguna forma de automatizar este proceso? Es decir, que yo levante los contenedores y realice también las migraciones de la base de datos.

Abre de nuevo http://localhost:5000. ¡Ahora sí!

Trabajando con CMD, entrypoints y commands
Piensa que al arrancar un contenedor Docker, siempre se ejecuta un único comando final. Lo que hacen ENTRYPOINT, CMD y command (de Compose) es decidir qué comando será ese y con qué argumentos.

CMD → “qué hacer por defecto”
Está en el Dockerfile.
Es el comando que Docker ejecuta si no se indica otra cosa. En nuestro caso:
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000", "--reload", "--debug"]
ENTRYPOINT → “qué se ejecuta siempre”
También va en el Dockerfile, o se puede sobreescribir en el docker-compose.yml.
Define el programa o script principal que siempre se ejecuta.
ENTRYPOINT ["/app/scripts/wait-for-db.sh"]
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
Normalmente, el entrypoint se combina con CMD en modo "eh, ejecuta esto (entrypoint) y luego este comando final (cmd)

Los entrypoints son muy útiles, sobre todo en los docker-compose's, porque puedo redefinir cómo quiero que inicie un contenedor en concreto SIN TENER QUE HACER UN BUILD DE NUEVO DE LA IMAGEN DE DOCKER. A diferencia de CMD, que viene predefinido, un entrypoint en mi docker-compose me da más versatilidad

command (de docker-compose.yml) → “sobrescribe el CMD”
En docker-compose.yml, el campo command: reemplaza al CMD del Dockerfile,pero no al ENTRYPOINT.

Combinación de elementos
Es importante entender cómo Docker trabaja con la orquestación y con los contenedores:

Lee el YAML como un diccionario
Ejecuta todo lo que haya en un Dockerfile EXCEPTUANDO directivas ENTRYPOINT y CMD
Monta el volumen o los volúmenes de ese contenedor (si procede)
Y ahora viene el detalle:

Si estamos levantando un contenedor a través de docker-compose, va a ejecutar los entrypoints y commands definidos en ese docker-compose.
Un entrypoint definido en un docker-compose va a sustituir a un entrypoint definido en el Dockerfile
Un command definido en un docker-compose va a sustituir a un CMD definido en el Dockerfile.
Como consejo:

Definimos un entrypoint en el docker-compose. Esto es así porque está fuera de la imagen de Docker y nos permite redefinir el inicio del servicio sin modificar la imagen
Eventualmente, definimos un CMD donde indicamos el comando final de ese contenedor al iniciar, que suele ser el arranque de Flask Server o de Gunicorn, dependiendo de si estamos en desarrollo o producción.
No obstante, como nuestros entrypoints son todos scripts en bash, podemos definir en el mismo el arranque final (comando final) y nos olvidamos de CMD.

Ejercicio 5: Volumen de trabajo
Problema de desarrollo
Vale, ya parece que le hemos pillado el tranquillo a Docker y a Docker Compose. Ahora vamos a hacer algo como desarrolladores. Ve al archivo "app/templates/base_templates.html" y cambia la línea 22:

<title>{{ FLASK_APP_NAME }} - Repository of feature models in UVL </title>
por:

<title>Hello world! </title>
Abre http://localhost:5000. Fíjate en el nombre de la pestaña del navegador, ¿ha cambiado? ¿No? Tsss, esto de los contenedores... bueno, va, vamos a reiniciarlo:

docker restart web_app_container
Abre http://localhost:5000. ¿Tampoco? Anda que vaya idea la de usar Docker... Claro, es a la hora de construir la imagen, hicimos "COPY . .", pero no hemos reconstruido nada, así que...

docker compose -f docker/docker-compose.yml build --no-cache web
docker compose -f docker/docker-compose.yml up -d --force-recreate web
Importante señalar que le pedimos que ignore la caché para que fuerce a copiar de nuevo todo el contenido (incluyendo el cambio de línea) al interior del contenedor.

Abre http://localhost:5000. ¡Ahora sí! Pero... ¿tengo que reconstruir TODO el contenedor en cada cambio línea? ¿Y eso es trabajar de forma ágil?

Crear volumen de trabajo (bind mount)
Evidentemente, eso no tiene sentido, tenemos que usar un volumen de trabajo que nos ayude a sincronizar el código de nuestro host con el código interno del contenedor. Luego en nuestro docker-compose:

  web:
    (....)
    ports:
      - "5000:5000"
    volumes:
      - ../:/app
    (....)
La línea

- ../:/app
Monta tu carpeta local del proyecto (../) dentro del contenedor, (/app). ¿Y por qué /app? Porque fue el espacio de trabajo que definimos en el Dockerfile. Como ves, la congruencia entre los Dockerfile y el docker-compose es importante.

Recarguemos de nuevo nustra configuración:

docker compose -f docker/docker-compose.yml up -d --build
Ahora ya podemos trabajar con el código en host sin tener que reconstruir la imagen a cada cambio de línea.

Cuidado con COPY y los bind mounts
Un bind mount (o “montaje de enlace”) es un tipo de volumen en el que Docker conecta directamente una carpeta de tu máquina (el host) con una carpeta dentro del contenedor.

Cuando usas un bind mount (por ejemplo ../:/app), Docker monta tu carpeta local sobre la ruta /app del contenedor. Eso significa que todo lo que estaba en /app dentro de la imagen desaparece al arrancar el contenedor, porque queda “oculto” por el volumen.

Ahora bien, durante la construcción de la imagen usamos la directiva COPY. ¿Por qué? Porque necesitamos el archivo requirements.txt para instalar las dependencias de Python del proyecto. Pero... ¿realmente necesitamos copiar TODO? No, claro que no, podemos optar por copiar exclusivamente aquellos archivos que nos hagan falta durante LA CONSTRUCCIÓN de la imagen. El bind mount luego lo machacará.

Muy importante: los bind mounts, al igual que cualquier otro volumen, están disponibles DESPUÉS de la construcción de la imagen. Es un error común creer que porque definamos un bind mount en un docker-compose, esos archivos aparecen ya para el contenedor. Esto es una fuente inagotable de problemas en la práctica.

Ejercicio 6: Distintas configuraciones de despliegue
Volvamos al punto de partida:

rm -r docker
mv docker.bk docker
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
Esta práctica ha sido engorrosa a propósito para quitar esa "magia" que existe cuando lanzas un comando simple de orquestación y todo se levanta y funciona perfectamente. Esa situación está muy bien para desarrollar, pero no para aprender.

Es importante que analices que existen distintos escenarios:

Hay un docker-compose para desarrollo.
Hay 3 docker-compose para producción.
Hay 6 Dockerfiles distintos
Hay 3 entrypoints distintos
Hay una carpeta llamada letsencrypt y otra llamada nginx. Dentro de nginx hay dos archivos de configuración dependiendo de si estamos en dev o en prod.
Autoevaluación
Estudia bien cada archivo. Tómate tu tiempo e intenta responder a estas preguntas:

¿Por qué en el docker-compose de dev NO se usa "ports" para enlazar con el host? (pista: nginx)
¿Qué es nginx y por qué es útil aquí?
¿Cuántos bind mounts usa el contenedor web en desarrollo? ¿Para qué sirve el bind mount del propio Docker?
¿Qué diferencia hay entre el entrypoint de desarrollo y el entrypoint de producción?
¿Por qué el servicio web en producción está montando todos esos bind mounts por separado? ¿No sería más sencillo hacer un volumen de todo el código?
Ejercicio propuesto
Levanta el entorno de Docker de desarrollo de uvlhub, siguendo la documentación oficial. Comprueba que puedes acceder a la app a través de localhost.
Intenta correr los test de Selenium https://docs.uvlhub.io/rosemary/testing/gui_tests#docker-environment-selenium-grid. Ten en cuenta que la forma de ejecutar y de visualizar los test de Selenium es distinta en un entorno Docker. Estudia bien la documentación, te hará falta para el desarrollo.
Comentarios finales
Cuanto más se parezca el entorno de desarrollo al de producción, menos errores habrá
En desarrollo es tentador usar configuraciones cómodas: servidor de Flask con --reload, bind mounts, base de datos vacía, logs en consola, etc. Pero cuanto más difiere ese entorno de producción, más probable es que algo falle al desplegar.

El código no falla por cambiar de servidor, sino porque los entornos se comportan distinto.

Ejemplos clásicos:

En desarrollo usas SQLite, en producción MariaDB → aparecen errores de compatibilidad SQL.
En desarrollo Flask sirve los archivos estáticos, en producción lo hace Nginx → las rutas cambian.
En desarrollo ejecutas flask run, en producción gunicorn → los hilos, workers o el reloader funcionan distinto.
Por eso, el objetivo ideal es que tu entorno de desarrollo (dev), de preproducción (staging) y de producción (prod) sean idénticos salvo por las credenciales.

Docker y docker-compose te permiten lograr eso: puedes usar la misma imagen base, cambiando solo el docker-compose (volúmenes, puertos, etc.).

Diferencia entre el servidor de Flask y Gunicorn
Flask trae incorporado un servidor de desarrollo. Es muy útil para depurar porque:

recarga automáticamente el código (--reload),
muestra errores detallados en el navegador,
solo ejecuta un proceso.
Sin embargo, no está pensado para producción:

no maneja múltiples conexiones concurrentes,
no tiene control de procesos ni balanceo de carga,
no implementa estándares de rendimiento o seguridad (como WSGI robusto).
Ahí entra Gunicorn (Green Unicorn), que sí es un servidor WSGI de producción (Web Server Gateway Interface, estándar de comunicación entre aplicaciones web Python y servidores web):

puede lanzar varios workers en paralelo,
maneja múltiples peticiones concurrentes,
se integra fácilmente con Nginx para servir contenido estático o HTTPS.