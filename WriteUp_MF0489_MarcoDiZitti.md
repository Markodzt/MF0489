# MF0489 | Examen Práctico   
## WriteUp Técnico – Máquina “Dr.Cifrado”  
### Autor: Marco Di Zitti  

---

# 1. Reconocimiento

En esta fase se realizaron tareas de reconocimiento activo y pasivo con el objetivo de identificar puertos, servicios y posibles vectores de ataque.

## 1.1 Herramientas utilizadas
- **Nmap** → Enumeración de puertos y servicios  
- **WPScan** → Enumeración de WordPress  
- **Hashcat** → Crack de hashes internos  
- **OpenSSL** → Descifrado de archivos Camellia  
- **Base64 / ROT13 / XOR** → Decodificación de cadenas ocultas  
- **DevTools / Inspect Element** → Análisis de JavaScript ofuscado  
- **Terminal de WordPress (plugin)** → Acceso interno al contenedor  
- **wget / curl** → Descarga de archivos y endpoints

## 1.2 Comandos ejecutados

### Escaneo inicial:
```bash
nmap -sC -sV -p- 10.10.116.128
```

### Resultado relevante:
- **22/tcp**   → open|filtered (SSH)
- **4444/tcp** → WordPress + PHP (lighttpd)
- **5353/tcp** → WordPress alternativo (login real)

### Identificación del CMS:
```bash
wpscan --url http://10.10.116.128:4444 --enumerate u
```

Resultado:
- WordPress **5.1.19**
- Usuario existente: **admin**
- XML-RPC habilitado

## 1.3 Análisis inicial de superficie de ataque

Tras revisar los archivos del plugin `intro-encoded.php`, se identificaron varias capas de cifrado y mecánicas tipo “puzzle”.  
Se descubrió la pista crítica `index.php/welcome`.

Siguiendo las rutas cifradas se obtuvieron:
- Archivos codificados (ROT13, Base64, XOR)
- Wordlist interna (`keys.txt`)
- Nuevas rutas del reto

---

# 2. Explotación

## 2.1 Archivos ocultos relevantes

En el archivo:

```
/wp-content/plugins/intro/_inc/cipherlogic-encoded.php
```

tras decodificación apareció:

```
The all-zero group char cannot appear in the alphabet
```

Indicando el uso de **Bacon Cipher**.

## 2.2 Puzzle en /cipherlogic

Ruta:

```
/index.php/cipherlogic
```

Puzzle resuelto mediante una secuencia obtenida tras varias capas de cifrado:

```
[2,1,5,7,1,4,8,3,3,2]
```

Tras completarlo se desbloquea:

```
/index.php/trituradora
```

La trituradora requería:
- Objeto → `U2FsdGVkX19PZWFwtJY7AjNnAoGv1uRHn7dcnbRNJEQ=`
- Potencia → `2157148332`

Resultado:

```
FLAG: admin
```

## 2.3 Descarga y descifrado de ZIP

Pista:

```
http://drcifrado.bs:4444/svyrf/3n1gm4.zip
```

Descifrado:

```bash
openssl -camellia-192-cbc -pbkdf2 -iter 1001 -d -in wp_pass -out wp_pass.txt
```

Contraseña obtenida:

```
1FItkRieW!km@v7fI3
```

WordPress real:

```
http://drcifrado.bs:5353/wp-login.php
```

Credenciales:
- admin
- 1FItkRieW!km@v7fI3

## 2.4 Plugin “No Works” → Shell interna

El plugin permitía descargar un archivo con hash:

```
no_works
```

Crack:

```bash
hashcat -m 0 hash.txt keys.txt
```

Contraseña obtenida:

```
bwcc2009new
```

Con esta contraseña se accede a una shell dentro del contenedor Docker.

---

# 3. Escalada de privilegios

## 3.1 Enumeración interna

Acceso con Terminal:

```
www-data@container:/var/www/html$
```

Confirmado:
- Sistema dentro de Docker
- Usuario limitado (www-data)
- Sin privilegios sudo
- Sin binarios SUID explotables

---

# 4. Flags encontradas
<img width="502" height="157" alt="Screenshot 2025-11-16 210033" src="https://github.com/user-attachments/assets/5babe215-1052-4be0-91c0-70550dea2876" />
<img width="940" height="422" alt="image-1" src="https://github.com/user-attachments/assets/2188530d-e47d-4b82-96df-c678e575bd5e" />
<img width="563" height="177" alt="image-2" src="https://github.com/user-attachments/assets/75f9109a-6380-4d14-b967-87b8c5d05966" />
<img width="940" height="483" alt="image" src="https://github.com/user-attachments/assets/0d17ff27-0b1b-49e7-9834-3890fa77881c" />


---

