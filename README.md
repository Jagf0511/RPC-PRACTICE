
# Práctica RPC en Docker – Arquitectura de Nube y Sistemas Distribuidos

**Autor:** Julián Andrés Guisao Fernández
**Materia:** Arquitectura de Nube y Sistemas Distribuidos  
**Entorno:** Máquina Virtual en Google Cloud

---

## Introducción

Esta práctica tiene como objetivo implementar un servicio remoto (RPC) usando Docker en una máquina virtual de Google Cloud. Se cubre la instalación de herramientas necesarias, creación del entorno, ejecución del servicio RPC y una explicación conceptual de su funcionamiento.

---

## Preparación del entorno

### 1. Instalación de Docker

```bash
sudo apt update && sudo apt upgrade
sudo apt install docker.io
```

### 2. Creación de carpeta de trabajo

```bash
mkdir RPC
cd RPC
```

### 3. Inicializar contenedor Ubuntu con carpeta RPC compartida

```bash
docker run -itd -v ./RPC:/rpc ubuntu bash
docker ps
docker exec -it <ID_CONTENEDOR> bash
```

Dentro del contenedor:

```bash
apt update && apt upgrade
hostid
apt install build-essential rpcbind libc6-dev net-tools inetutils-ping nano libtirpc-dev
```

### 4. Iniciar servicio RPC

```bash
service rpcbind start
```

Si aparece un error, se soluciona con:

```bash
mkdir -p /run/sendsigs.omit.d/rpcbind
service rpcbind start
```

Verificamos puertos activos:

```bash
netstat -tulpan
```

---

## Configuración de usuario

Creamos un nuevo usuario:

```bash
useradd -m id427265 -s /bin/bash
su - id427265
```

Creamos la carpeta `.x` donde irá el código RPC.

---

## Recordatorio de uso de punteros

Cuando una variable es un puntero a una estructura, para acceder a sus miembros se usa `->` en lugar de `.`.

Ejemplo:

```c
estructura->campo
```

---

## RPC: Comunicación en dos fases

RPC (Remote Procedure Call) funciona en dos fases:

1. **Cliente realiza una petición al servidor.**
2. **Servidor ejecuta la función y responde.**

Esto permite ejecutar funciones en otra máquina como si fueran locales.

---

# Ejercicio RPC: Suma de hasta 10 enteros desde la línea de comandos

## Requerimiento

Diseñar un programa RPC que permita sumar hasta 10 enteros proporcionados desde la línea de comandos. Esta versión amplía el ejemplo visto en clase, que solo sumaba dos valores.

---

## Paso 1: Definición del archivo `.x`

Creamos el archivo `archivo.x` con el siguiente contenido:

```c
struct sumandos {
    int sumando1;
    int sumando2;
};

program PROGRAMA_SUMA {
    version VERSION_SUMA {
        int suma(sumandos) = 1;
        int resta(sumandos) = 2;
    } = 1;
} = 0x20000001;
```

### Explicación

- `struct sumandos`: estructura básica con dos enteros.
- `program` y `version`: se utilizan para identificar de forma única los procedimientos en `rpcbind`.
- Cada procedimiento (suma y resta) tiene un número asociado.

---

## Paso 2: Generar código fuente

Generamos todos los archivos necesarios automáticamente:

```bash
rpcgen -a archivo.x
```

---

## Paso 3: Implementación del servidor (`archivo_server.c`)

```c
#include "archivo.h"

int *suma_1_svc(sumandos *argp, struct svc_req *rqstp) {
    static int result;
    result = argp->sumando1 + argp->sumando2;
    return &result;
}

int *resta_1_svc(sumandos *argp, struct svc_req *rqstp) {
    static int result;
    result = argp->sumando1 - argp->sumando2;
    return &result;
}
```

---

## Paso 4: Implementación del cliente (`archivo_client.c`)

```c
void programa_suma_1(char *host) {
    CLIENT *clnt;
    int *result_1;
    sumandos suma_1_arg;
    int *result_2;
    sumandos resta_1_arg;

    suma_1_arg.sumando1 = 3;
    suma_1_arg.sumando2 = 2;

    resta_1_arg.sumando1 = 3;
    resta_1_arg.sumando2 = 5;

#ifndef DEBUG
    clnt = clnt_create(host, PROGRAMA_SUMA, VERSION_SUMA, "udp");
    if (clnt == NULL) {
        clnt_pcreateerror(host);
        exit(1);
    }
#endif

    result_1 = suma_1(suma_1_arg, clnt);
    if (result_1 == (int *) NULL) {
        clnt_perror(clnt, "call failed");
    }

    result_2 = resta_1(resta_1_arg, clnt);
    if (result_2 == (int *) NULL) {
        clnt_perror(clnt, "call failed");
    }

    printf("la suma %d\n", *result_1);
    printf("la resta %d\n", *result_2);

#ifndef DEBUG
    clnt_destroy(clnt);
#endif
}
```

---

## Paso 5: Makefile personalizado (`Makefile.archivo`)

```makefile
# Compiler flags
CFLAGS += -g -D_REENTRANT -I/usr/include/tirpc
LDLIBS += -lpthread -ltirpc
RPCGENFLAGS = -M
```

Compilamos con:

```bash
make -f Makefile.archivo
```

---

## Paso 6: Ejecución de servidor y cliente

### En la primera terminal (dentro del contenedor):

```bash
./archivo_server
```

### En la segunda terminal:

```bash
docker exec -it <ID_CONTENEDOR> bash
cd /ruta/a/la/carpeta/RPC
./archivo_client 127.0.0.1
```

---

## Limitaciones de RPC

- No es escalable (orientado a LAN).
- No está diseñado para redes WAN.
- Cualquier cambio requiere recompilación (archivo `.x` y Makefile).
- Solo se comunica entre dos procesos claramente definidos (cliente y servidor).

---

## Conclusión

Esta práctica refuerza el entendimiento de RPC desde la preparación del entorno hasta la ejecución de servicios personalizados.


# Extensión del Ejercicio: Suma de un vector de hasta 10 enteros por línea de comandos

## Objetivo

Diseñar un servicio RPC que reciba hasta 10 enteros como entrada por línea de comandos, los envíe al servidor, y este devuelva la suma total.

---

## Paso 1: Definir archivo `.x` para vectores

Creamos o modificamos `archivo_vector.x`:

```c
const MAX = 10;

struct vector {
    int datos[MAX];
    int cantidad;
};

program PROGRAMA_VECTOR {
    version VERSION_VECTOR {
        int suma_vector(vector) = 1;
    } = 1;
} = 0x20000002;
```

**Explicación:**
- `MAX = 10` define el tamaño máximo del vector.
- `struct vector` contiene un arreglo `datos` y un entero `cantidad` para saber cuántos datos se usan realmente.
- El procedimiento `suma_vector` tomará esta estructura y devolverá la suma de los enteros.

---

## Paso 2: Generar código base

```bash
rpcgen -a archivo_vector.x
```

---

## Paso 3: Implementación del servidor (`archivo_vector_server.c`)

```c
#include "archivo_vector.h"

int *suma_vector_1_svc(vector *argp, struct svc_req *rqstp) {
    static int result;
    result = 0;

    for (int i = 0; i < argp->cantidad; i++) {
        result += argp->datos[i];
    }

    return &result;
}
```

---

## Paso 4: Cliente (`archivo_vector_client.c`)

```c
void programa_vector_1(char *host, int argc, char *argv[]) {
    CLIENT *clnt;
    int *result;
    vector entrada;

    entrada.cantidad = argc - 2; // primer arg es el nombre del ejecutable, segundo es host

    if (entrada.cantidad > 10) {
        printf("Máximo 10 sumandos permitidos.\n");
        exit(1);
    }

    for (int i = 0; i < entrada.cantidad; i++) {
        entrada.datos[i] = atoi(argv[i + 2]); // los datos comienzan desde argv[2]
    }

    clnt = clnt_create(host, PROGRAMA_VECTOR, VERSION_VECTOR, "udp");
    if (clnt == NULL) {
        clnt_pcreateerror(host);
        exit(1);
    }

    result = suma_vector_1(entrada, clnt);
    if (result == (int *) NULL) {
        clnt_perror(clnt, "call failed");
    } else {
        printf("Resultado de la suma: %d\n", *result);
    }

    clnt_destroy(clnt);
}
```

### Para compilar y ejecutar:

1. Edita el `Makefile` y asegúrate de usar `archivo_vector` en las reglas.
2. Compila:

```bash
make -f Makefile.archivo
```

3. Ejecuta en la primera terminal:

```bash
./archivo_vector_server
```

4. Ejecuta en otra terminal con sumandos desde CLI:

```bash
./archivo_vector_client 127.0.0.1 1 2 3 4 5
```

---

## Conclusión

Este ejemplo permite una operación más flexible y dinámica mediante el uso de arreglos, simulando cómo se enviarían múltiples datos en una llamada remota. Es fundamental declarar correctamente la cantidad de elementos para evitar lecturas incorrectas.
