# App Diccionarios

## Objetivo

Será una App Web. 
Nos vamos a centrar en este documento en el backend.
Va a ser un microservicio(s) que exponga una API REST para trabajar con diccionarios de palabras... y hacer búsquedas de palabras y otras funciones que puedan ser de interés.

## Actores y Casos de Uso

### Diagrama general de casos de uso
```plantuml
@startuml
left to right direction
actor "Usuario" as U
actor "Editor" as E
actor "Administrador" as A

U --> (Consultar palabra)
E --> (Gestionar diccionarios)
A --> (Gestionar usuarios y editores)
A --> (Importar/Exportar diccionarios)
@enduml
```


### Analicemos más despacio a nuestro Usuario

```plantuml
@startuml
left to right direction
actor "Usuario" as U

rectangle "App Diccionarios" {
    package "Consulta de palabras" {
        U --> (Búsqueda exacta)
        U --> (Búsqueda por prefijo)
        U --> (Búsqueda por similitud)
    }
}
@enduml
```

#### Comentarios/Observaciones:

> La búsqueda exacta, quién me la puede resolver bien?
Base de datos relacional? SI

> La búsqueda por prefijo, quién me la puede resolver bien?
Base de datos relacional? SI:  LIKE %

> Busqueda por similitud? 
Necesito herramientas especializadas: 
- Elasticsearch (indexador)
   Quiero buscar la palabra     cocreta --> croqueta
                                ierva   --> hierba
- Oracle: Oracle Text

### Analicemos más despacio a nuestro Editor

```plantuml 
@startuml
left to right direction
actor "Editor" as E

rectangle "App Diccionarios" {
    package "Gestión de diccionarios" {
        E --> (Gestión palabras)
        E --> (Gestión de idiomas)
        E --> (Gestión de diccionarioS)
    }
}
@enduml
```

### Analicemos más despacio a nuestro Administrador

```plantuml
@startuml
left to right direction
actor "Administrador" as A  

rectangle "App Diccionarios" {
    package "Administración de usuarios" {
        A --> (Gestión de usuarios)
    }
    package "Carga de diccionarios" {
        A --> (Importar diccionario)
        A --> (Exportar diccionario)
    }
}
@enduml
```

### Si no es muy complejo, incluso podría juntarlo todo en un solo diagrama

```plantuml
@startuml
left to right direction
actor "Usuario" as U
actor "Editor" as E
actor "Administrador" as A

rectangle "App Diccionarios" {
    package "Consulta de palabras" {
        U --> (UC-U-001: Búsqueda exacta)
        U --> (UC-U-002: Búsqueda por prefijo)
        U --> (UC-U-003: Búsqueda por similitud)
    }
    package "Gestión de idiomas/diccionarios/palabras" {
        E --> (UC-E-001: Gestión palabras)
        E --> (UC-E-002: Gestión de idiomas)
        E --> (UC-E-003: Gestión de diccionarios)
    }
    package "Administración de usuarios" {
        A -up-> (UC-A-001: Gestión de usuarios)
    }
    package "Carga de diccionarios" {
        A -up-> (UC-A-002: Importar diccionario)
        A -up-> (UC-A-003: Exportar diccionario)
    }
}
@enduml
```

#### Roles:
- Administrador
- Editor
- Usuario

#### Entidades:
- Diccionario
- Palabra
- Definiciones
- Ejemplos
- Categoria gramatical
- Pronunciación
- Idioma

## Requisitos funcionales / No funcionales

- UC-U-001: Búsqueda exacta
    - RF-001: El sistema permitirá a los usuarios buscar palabras exactas en un diccionario, de un idioma específico.
    - RF-002: El sistema devolverá las definiciones, con ejemplos y categoria gramatical, pronunciación de la palabra buscada.
    - RF-003: El sistema debe permitir reproducir la pronunciación de la palabra.
... Me puedo pasar rato.

---

Pronunciación de la palabra: AUDIO.

> Dónde lo guardo? BBDD Relacional? Poder puedo... BLOB.. pero esto es ruina.
> Posiblemente me interese un almacenamiento orientado a objetos: S3, MinIO, etc.

## Arquitectura de la información

Aquí vamos a entrar más en detalle de esas entidades:

### Diagrama entidad-Relación ... existe esto en UML?

En UML y en POO se considera las entidades son un tipo de clases.

```mermaid
classDiagram

    class Idioma {
        +String id
        +String nombre
    }

    class Diccionario {
        +String id
        +String nombre
        +String idioma
    }

    class Palabra {
        +String id
    }

    class Definicion {
        +String id
        +String texto
    }

    class Ejemplo {
        +String id
        +String texto
    }

    class CategoriaGramatical {
        +String id
        +String nombre
    }

    class Pronunciacion {
        +String id
        +String urlAudio
    }

    class Etiquetas {
        +String id
        +String nombre
    }

    Idioma "1" -- "0..*" Diccionario : contiene 
    Diccionario "1" -- "0..*" Palabra : contiene 
    Palabra "1" -- "0..*" Definicion : tiene 
    Definicion "1" -- "0..*" Ejemplo : incluye 
    CategoriaGramatical "1" -- "0..*" Definicion : clasifica 
    Pronunciacion "1" -- "1" Palabra : corresponde a 
    Etiquetas "0..*" -- "0..*" Definicion : etiqueta 
    
```

> Notas: 

Los Ids de las entidades, pueden ser UUIDs o códigos (ejemplo: Idioma Español: ES) (otra cosa es que de cara a una BBDD relacional, usemos enteros autoincrementales adicionalmente para favorecer las relaciones).

## Arquitectura del sistema (Backend)


BBDD para guardar datos estructurados: PostgreSQL
Indexador para búsquedas por similitud: Elasticsearch
Almacenamiento de ficheros (audio pronunciaciones): MinIO / S3

Backend:
    Controlador API REST gestion-usuarios
    Controlador API REST import-export-diccionarios
    Controlador API REST gestion-diccionarios
    Controlador API REST consulta-palabras

### Diagrama de componentes / paquetes palabras

Puedo comenzar haciendo un modelado TOP-BOTTOM (controlador) o un BOTTOM-UP (dominio)

```mermaid
graph TD
    A[API Repositorio de <br/>Palabras/Diccionarios/Idiomas]
    B[Repositorio PostgresQL de <BR/>Palabras/Diccionarios/Idiomas]

    L[API Gestor de Indexación de <BR/>Palabras/Diccionarios/Idiomas]
    M[Gestor de Indexación Elasticsearch de <BR/>Palabras/Diccionarios/Idiomas]

    N[API Gestor de Almacenamiento de <BR/>Pronunciaciones]
    O[Gestor de Almacenamiento MinIO de <BR/>Pronunciaciones]

    C[API Servicio de <br/>Búsqueda de <BR/> Palabras/Diccionarios/Idiomas]
    D[API Servicio de <br/>Gestión de <BR/> Palabras/Diccionarios/Idiomas]

    E[Servicio de <br/>Búsqueda de <BR/> Palabras/Diccionarios/Idiomas]
    F[Servicio de <br/>Gestión de <BR/> Palabras/Diccionarios/Idiomas]

    G[API Controlador REST de <br/>Consulta de <BR/> Palabras/Diccionarios/Idiomas]
    H[API Controlador REST de <br/>Gestión de <BR/> Palabras/Diccionarios/Idiomas]

    I[Controlador REST V1 de <br/>Consulta de <BR/> Palabras/Diccionarios/Idiomas]
    J[Controlador REST V1 de <br/>Gestión de <BR/> Palabras/Diccionarios/Idiomas]

    K[Backend App Diccionarios]

    M --> L
    O --> N
    D --> L
    D --> N
    E --> L
    B --> A
    E --> C
    F --> D
    E --> A
    F --> A
    I --> C
    J --> D
    I --> G
    J --> H

    K --> I
    K --> J
    K --> B
    K --> E
    K --> F
    K --> M
    K --> O

```

```xml
<!-- pom.xml dependencias -->
<modules>
    <module>api-repositorio-palabras</module>
    <module>repositorio-postgresql-palabras</module>
    <module>api-servicio-busqueda-palabras</module>
    <module>api-servicio-gestion-palabras</module>
    <module>servicio-busqueda-palabras</module>
    <module>servicio-gestion-palabras</module>
    <module>api-controlador-rest-consulta-palabras</module>
    <module>api-controlador-rest-gestion-palabras</module>
    <module>controlador-rest-v1-consulta-palabras</module>
    <module>controlador-rest-v1-gestion-palabras</module>
    <module>app-diccionarios-backend</module>
</modules>

<!-- pom.xml del app-diccionarios-backend -->
<dependencies>
    <dependency>
        <groupId>com.ejemplo</groupId>
        <artifactId>controlador-rest-v1-consulta-palabras</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.ejemplo</groupId>
        <artifactId>controlador-rest-v1-gestion-palabras</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.ejemplo</groupId>
        <artifactId>repositorio-postgresql-palabras</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.ejemplo</groupId>
        <artifactId>servicio-busqueda-palabras</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.ejemplo</groupId>
        <artifactId>servicio-gestion-palabras</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>

<!-- pom.xml del controlador-rest-v1-consulta-palabras -->
<dependencies>
    <dependency>
        <groupId>com.ejemplo</groupId>
        <artifactId>api-controlador-rest-consulta-palabras</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.ejemplo</groupId>
        <artifactId>api-servicio-busqueda-palabras</artifactId>
        <version>1.0.0</version>
    </dependency>
```

```java

public interface IdiomasService {
    List<Idioma> obtenerIdiomas();
    Idioma obtenerIdiomaPorId(String id);
    Idioma crearIdioma(Idioma idioma);
    Idioma actualizarIdioma(String id, Idioma idioma);
    void eliminarIdioma(String id);
}

public interface DiccionariosService {
    List<Diccionario> obtenerDiccionarios();
    Diccionario obtenerDiccionarioPorId(String id);
    List<Diccionario> obtenerDiccionariosPorIdioma(String idiomaId);
    Diccionario crearDiccionario(Diccionario diccionario);
    Diccionario actualizarDiccionario(String id, Diccionario diccionario);
    void eliminarDiccionario(String id);
}

public interface GestionPalabrasService {
    List<Palabra> obtenerPalabrasPorDiccionario(String diccionarioId);
    Palabra agregarPalabra(String diccionarioId, Palabra palabra);
    Palabra actualizarPalabra(String id, Palabra palabra);
    void eliminarPalabra(String id);
}
public interface BusquedaPalabrasService {
    List<Palabra> buscarPalabrasExactasPorDiccionario(String diccionarioId, String termino);
    List<Palabra> buscarPalabrasPorPrefijoPorDiccionario(String diccionarioId, String prefijo);
    List<Palabra> buscarPalabrasPorSimilitudPorDiccionario(String diccionarioId, String termino);
    List<Palabra> buscarPalabrasExactasPorIdioma(String idiomaId, String termino);
    List<Palabra> buscarPalabrasPorPrefijoPorIdioma(String idiomaId, String prefijo);
    List<Palabra> buscarPalabrasPorSimilitudPorIdioma(String idiomaId, String termino);
}
```
#### Capa dominio

##### Diagrama de componentes API Del Repositorio de Palabras/Diccionarios/Idiomas

###### Modelos del API Repositorio de Palabras/Diccionarios/Idiomas
```mermaid
classDiagram

    class Idioma {
        <<interface>> 
        +String id
        +String nombre
    }

    class Diccionario {
        <<interface>> 
        +String id
        +String nombre
        +String idioma
    }

    class DatosModificarDiccionario {
        <<interface>> 
        +String nombre
    }

    class Palabra {
        <<interface>>
        +String id
    }

    class Definicion {
        <<interface>>
        +String id
        +String texto
    }

    class Ejemplo {
        <<interface>>
        +String id
        +String texto
    }

    class CategoriaGramatical {
        <<interface>>
        +String id
        +String nombre
    }

    class Etiquetas {
        <<interface>>   
        +String id
        +String nombre
    }

    Idioma "1" -- "0..*" Diccionario : contiene
    Diccionario "1" -- "0..*" Palabra : contiene
    Palabra "1" -- "0..*" Definicion : tiene
    Definicion "1" -- "0..*" Ejemplo : incluye
    CategoriaGramatical "1" -- "0..*" Definicion : clasifica
    Etiquetas "0..*" -- "0..*" Definicion : etiqueta
```
##### Diagrama de clases API Repositorio PostgreSQL de Palabras/Diccionarios/Idiomas

```mermaid

classDiagram

    class IdiomasRepositorio {
        +List<Idioma> obtenerIdiomas()
        +Idioma obtenerIdiomaPorId(String id)
        +Idioma crearIdioma(Idioma idioma)
        +Idioma actualizarIdioma(String id, Idioma idioma)
        +void eliminarIdioma(String id)
    }

    class DiccionariosRepositorio {
        +List<Diccionario> obtenerDiccionarios()
        +Diccionario obtenerDiccionarioPorId(String id)
        +List<Diccionario> obtenerDiccionariosPorIdioma(String idiomaId)
        +Diccionario crearDiccionario(Diccionario diccionario)
        +Diccionario actualizarDiccionario(String id, DatosModificarDiccionario diccionario)
        +void eliminarDiccionario(String id)
    }

    class PalabrasRepositorio {
        +List<Palabra> obtenerPalabrasPorDiccionario(String diccionarioId)
        +Palabra agregarPalabra(String diccionarioId, Palabra palabra)
        +Palabra actualizarPalabra(String id, Palabra palabra)
        +void eliminarPalabra(String id)
    }

    class ArgumentoInvalidoExcepcion {
        +String mensaje
    }

    class RepositoryInternalExcepcion {
        +String mensaje
    }

    IdiomasRepositorio ..> ArgumentoInvalidoExcepcion : throws
    IdiomasRepositorio ..> RepositoryInternalExcepcion : throws

```


#### Diagrama de secuencia para el backend de búsqueda similar de una palabra

```mermaid
sequenceDiagram
    participant U as Usuario
    participant C as Controlador REST V1 Consulta Palabras
    participant S as Servicio Búsqueda Palabras
    participant S2 as Servicio Monitorización
    participant R as Repositorio Palabras
    participant B as BBDD PostgreSQL
    participant I as Elasticsearch

    U->> +C: Solicitar búsqueda similar de palabra "cocreta" en diccionario "ES-001"
    C->> +S: buscarPalabrasPorSimilitudPorDiccionario("ES-001", "cocreta")
    S->> +I: Consultar Elasticsearch con término "cocreta"
    I--x -S: Devolver lista de palabras similares ["croqueta", "coreta", "corcheta"]
    S->> +R: Obtener detalles de palabras similares
    R->> +B: Consultar PostgreSQL con IDs de palabras similares
    B--x -R: Devolver detalles de palabras similares
    R--x -S: Devolver lista de objetos Palabra
    S--x C: Devolver lista de palabras similares con detalles
    S-) -S2: Registrar búsqueda
    C--x -U: Responder con lista de palabras similares en JSON
```

---

Al montar un sistema, cada vez más nos centramos en la experiencia de usuario (UX).
Hacemos un diseño User-Centered Design (UCD).

En el mundo de la UX y del UCD hay disntas técnicas / metodologías para entender mejor a los usuarios y sus necesidades.
- Diagrama de Garrett. 5 capas:
    - Necesidades del usuario ~ Diagrama de casos de uso
    - Especificación Funcional ~ Requisitos
    - Estructura
      - Diseño de la interacción
      - Arquitectura de la información
    - Esqueleto (planteamos la UI)
    - Superficie (UI)
- Honeycomb de Peter Morville





---

# Programación REACTIVA

Esta estudiada y definida en una especie de estándar: Reactive Streams (https://www.reactive-streams.org/)

Hay un monton de implementaciones de Reactive Streams en distintos lenguajes de programación.
JS -> RxJS
Java -> Project Reactor, Spring WebFlux, Akka Streams

Es un tipo de programación asíncona, no bloqueante [ y orientada a flujos de datos (streams) ].

Programación asíncrona:
- Bloqueante
- No bloqueante
  - Con callbacks
  - Fire-and-forget

En ocasiones:

    Tengo el controlador REST
        Llama al servicio de forma asíncrona
        Deja anotado lo que hay que hacer cuando llegue la respuesta
        Y el acaba!

SpringWeb HTTP REST

Tomcat, Jetty, WebLoop -> Bloqueante
   Cada petición abre un hilo. Y puedo habrir cientos de hilos... pero no miles!
   Y eso limita el número de peticiones concurrentes que puedo atender.

Webflux / Netty -> No bloqueante
   Cada petición no abre un hilo. Usa un pool de hilos cerrado... y las tareas se dejan en pausa hasta que llegan las respuestas.
   Puedo atender miles de peticiones concurrentes.

llamarAsincronamente() -> Future/Promise     Vale por una camisa

unFuture.get()  // Bloqueante

await unPromise // Bloqueante

unPromise.then( 
    res => {
        // Hacer algo con res
    }
) // No bloqueante // Reactive

JAVA 8 --> java.util.function :: ->

















---

# Máquina de café

Diagramas de casos de uso
Diagramas de secuencia
Diagramas de actividades
## Diagramas de estados

El modelo de máquinas de estados es un modelo matemático que describe el comportamiento de un sistema en términos de estados y transiciones entre esos estados.

Hay modelos muy tradicionales. UML definió su propio modelo de máquinas de estados basado en los modelos tradicionales pero con algunas extensiones.
Muchas herramiientas y librerías de software permiten definir máquinas de estados basadas en UML.
Spring State Machine es una librería de Java que permite definir máquinas de estados basadas en UML.

- State: Representa un estado en el que puede estar el sistema.
- Transition: Representa una transición entre dos estados, que ocurre en respuesta a un evento o condición (automáticos).
- Guard: Condición que debe cumplirse para que una transición pueda ocurrir.
- Efect/Action: Acción que se ejecuta cuando se produce una transición.
- Context: Conjunto de propiedades que pueden influir en el comportamiento de la máquina de estados.

Dentro de las máquinas de estados UML se habla de:
- Estados jerárquicos: Estados que contienen otros estados, permitiendo una organización en niveles.
- Estados paralelos:   Esto es raro... ya que rompe con el principio básico de una máquina de estados (un único estado activo en cada momento)

Componentes Web -> Máquinas de estados


                                    Sistema Backend
                                        Sesión de formación
                                           Pendiente de inicio
                                           Abierta para registro
                                           Espera
                                           Realizando pregunta
                                           Espera de respuesta
                                           Pregunta finalizada y Visualización de estadísticas
                                           Cerrada
Preguntas > App movil QR

    Pedido en una web de pedidos
        Carrito de la compra
        Pago
        Confirmación del pedido
        Preparación del pedido
        Envío del pedido
        Entrega del pedido




---


Tienda de Animalitos Fermín

    Alta de un animalito:
        Nombre,
        Tipo de animalito (perro, gato, pez, pájaro, roedor, reptil, etc.),
        Fecha de nacimiento
        Descripción
        Foto
    
    Datos de un animalito:
        ID PUBLICO
        Nombre,
        Tipo de animalito (perro, gato, pez, pájaro, roedor, reptil, etc.),
        Fecha de nacimiento
        Descripción
        Foto
        Estado (disponible, reservado, adoptado)
    
    Modificar datos de un animalito
        Tipo de animalito (perro, gato, pez, pájaro, roedor, reptil, etc.),
        Descripción
        Foto
        Estado

    Funciones:
        Animalito altaAnimalito(DatosDeAltaDeAnimalito)
        Animalito modificarAnimalito(ID_PUBLICO, DatosDeModificacionDeAnimalito)
        Animalito recuperarAnimalito(ID_PUBLICO)
        Animalito eliminarAnimalito(ID_PUBLICO)

SOLID:
    I: Interface Segregation Principle
        Muchas interfaces específicas mejor que una interfaz general.


---
