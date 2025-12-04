# Documentación de Arquitectura de la Aplicación de Diccionarios (modo consola)

Este documento describe la **arquitectura a nivel de paquetes/módulos** de una aplicación de diccionarios en **modo consola**.  
El objetivo es mostrar cómo se separan responsabilidades y cómo se aplican principios de diseño como **Separation of Concerns (SoC)** y **Dependency Inversion Principle (DIP)**.

No es un diagrama de clases, sino una **vista de arquitectura / paquetes**: qué bloques existen y cómo se relacionan.

---

## Vista de Arquitectura (Diagrama de paquetes UML en Mermaid)

```mermaid
---
title: Arquitectura por paquetes - Aplicación de Diccionarios (Consola)
---
graph LR

    subgraph UI[Paquete: Interfaz de Usuario]
        UI_API[InterfazDeUsuarioAPI<br/>&lt;&lt;API&gt;&gt;]
        UI_CONS[InterfazDeUsuarioConsola<br/>&lt;&lt;impl&gt;&gt;]
    end

    subgraph APP[Paquete: Aplicación / Flujo]
        APP_MAIN[Aplicacion<br/>&lt;&lt;app&gt;&gt;]
    end

    subgraph NEGOCIO[Paquete: Gestión de Palabras]
        GP_API[GestorDePalabrasAPI<br/>&lt;&lt;API&gt;&gt;]
        GP_FICH[GestorDePalabrasEnFicheros<br/>&lt;&lt;impl&gt;&gt;]
    end

    subgraph DATOS[Paquete: Datos de Diccionarios]
        DIC[DiccionariosConcretos<br/>&lt;&lt;datos&gt;&gt;]
    end

    %% Implementaciones que realizan las APIs
    UI_CONS --> UI_API
    GP_FICH --> GP_API

    %% La aplicación orquesta usando solo APIs (DIP)
    APP_MAIN --> UI_API
    APP_MAIN --> GP_API

    %% Acceso a datos desde la implementación de negocio
    GP_FICH --> DIC
````

---

## Descripción de los paquetes

### Paquete `Interfaz de Usuario`

Contiene **todo lo relacionado con hablar con el usuario** (entrada/salida), separado en:

* `InterfazDeUsuarioAPI` (`<<API>>`)

  * Define **qué** necesita la aplicación de la UI:

    * Pedir idioma.
    * Pedir palabra.
    * Mostrar significados.
    * Mostrar mensajes informativos/errores.
  * No sabe nada de consola, ni de Swing, ni de HTML. Solo define **operaciones**.

* `InterfazDeUsuarioConsola` (`<<impl>>`)

  * Implementación concreta que usa la **consola**:

    * Lee de `System.in`, escribe en `System.out`, etc.
  * Depende de `InterfazDeUsuarioAPI` porque la **implementa**.
  * En un futuro puedes añadir:

    * `InterfazDeUsuarioDesktop`, `InterfazDeUsuarioWeb`, etc., sin tocar el resto de capas.

**Idea clave**: la aplicación general habla con una **API de UI**, no con “la consola” directamente.

---

### Paquete `Aplicación / Flujo`

* `Aplicacion` (`<<app>>`)

Es el **cerebro del caso de uso** “buscar una palabra en un diccionario”:

1. Pide el idioma al usuario a través de `InterfazDeUsuarioAPI`.
2. Comprueba, vía `GestorDePalabrasAPI`, si hay diccionario para ese idioma.
3. Si no hay:

   * Muestra un mensaje al usuario y termina.
4. Si hay:

   * Pide la palabra al usuario.
   * Llama a `GestorDePalabrasAPI` para buscarla.
   * Si no existe, avisa al usuario.
   * Si existe, muestra los significados.

**Muy importante**:

* Aquí **no hay acceso a ficheros**.
* Aquí **no hay lógica de parseo** de diccionarios.
* Aquí **no se conoce la consola**.
* Solo se depende de **APIs**: `InterfazDeUsuarioAPI` y `GestorDePalabrasAPI`.

---

### Paquete `Gestión de Palabras`

Agrupa la lógica de **negocio** (no de UI, no de infraestructura):

* `GestorDePalabrasAPI` (`<<API>>`)

  * Define operaciones como:

    * `existeIdioma(idioma)`.
    * `buscarPalabra(idioma, palabra)`.
    * `obtenerSignificados(idioma, palabra)`.
  * No se mete en cómo se almacenan ni cómo se recuperan los datos.
  * Es la “cara” del backend de diccionarios.

* `GestorDePalabrasEnFicheros` (`<<impl>>`)

  * Implementación concreta que:

    * Lee de ficheros tipo:

      ```text
      diccionarios/
        ES.txt
        EN.txt
        FR.txt
        ...
      ```
    * Entiende el formato:

      ```text
      manzana=Fruto del manzano...
      melon=Fruto del melonar...|Persona con pocas luces.
      ```
  * Depende de:

    * `GestorDePalabrasAPI` (porque la implementa).
    * `DiccionariosConcretos` (porque ahí están los datos reales).

Si mañana decides usar BBDD o un servicio REST:

* Creas `GestorDePalabrasEnBBDD` o `GestorDePalabrasViaREST` que implementen **la misma API**.
* La aplicación y la UI no cambian: solo cambias qué implementación se “inyecta”.

---

### Paquete `Datos de Diccionarios`

* `DiccionariosConcretos` (`<<datos>>`)

Contiene los **datos físicos**:

  diccionarios/
    ES.txt
    EN.txt
    FR.txt
    IT.txt
    DE.txt


y/o scripts, recursos, ficheros de configuración, etc.

Características:

* Cambia mucho más a menudo (nuevos idiomas, nuevas palabras, nuevas versiones).
* No debe contener lógica de negocio.
* Solo lo toca la implementación de `GestorDePalabrasAPI`.

---

## Perspectiva de arquitectura

### Separation of Concerns (SoC)

* **UI** (cómo interactúo con el humano)
  → Paquete `Interfaz de Usuario`.

* **Aplicación / flujo de casos de uso** (qué pasos sigo para resolver el problema)
  → Paquete `Aplicación / Flujo`.

* **Negocio de diccionarios** (qué es “buscar palabra”, qué es “idioma soportado”)
  → Paquete `Gestión de Palabras`.

* **Datos** (dónde están las palabras físicamente)
  → Paquete `Datos de Diccionarios`.

Cada problema en su caja. Menos acoplamiento, más claridad.

---

### Dependency Inversion Principle (DIP)

Los módulos de alto nivel (**Aplicacion**) dependen de **abstracciones**:

* `Aplicacion --> InterfazDeUsuarioAPI`
* `Aplicacion --> GestorDePalabrasAPI`

Las implementaciones concretas dependen de esas APIs y de detalles:

* `InterfazDeUsuarioConsola --> InterfazDeUsuarioAPI`
* `GestorDePalabrasEnFicheros --> GestorDePalabrasAPI`
* `GestorDePalabrasEnFicheros --> DiccionariosConcretos`

Resultado:

* Puedes cambiar **UI** sin tocar negocio.
* Puedes cambiar **cómo almacenas diccionarios** sin tocar la UI ni el flujo.
* El sistema está preparado para crecer y versionarse sin convertirse en un monolito inamovible.
