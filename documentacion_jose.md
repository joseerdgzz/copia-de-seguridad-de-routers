# Extracción de configuraciones con usuario operador

## 1. Escenario

### 1.1 Usuario operador para hacer copias

**Datos de inicio de sesión:**

- Usuario: `operador`
- Contraseña: `Oper1234`

## 2. Preparación del equipo Ubuntu

### 2.1 Carpeta de destino

mkdir -p ~/configs-monitorización
cd ~/configs-operador

### 2.2 Cliente SSH

El Cisco y el Olive usan algoritmos antiguos que el cliente SSH de Ubuntu tiene desactivados.


cat >> ~/.ssh/config << 'EOF'
Host 10.10.10.*
    KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
    StrictHostKeyChecking accept-new
EOF
chmod 600 ~/.ssh/config

### 2.3 Procedimiento común en cada router

1. Entrar como administrador y crear el usuario operador con permisos de solo lectura.
2. Guardar la configuración.
3. Desde Ubuntu, conectarse como operador y volcar la configuración a un fichero.

## 3. Cisco IOS (10.10.10.2)

### 3.1 Limitación de los permisos

Cisco usa niveles de privilegio del 0 al 15: el 1 es el modo usuario básico (`Router>`) y el 15 el administrador (`Router#`). Se crea el operador en el nivel 5 y se le permite un único comando extra: show startup-config.

### 3.2 Configuración como administrador

enable
configure terminal
username operador privilege 5 secret Oper1234
privilege exec level 5 show startup-config
line vty 0 4
 login local
 transport input ssh
end
write memory

Show running-config solo muestra las líneas que ese nivel podría configurar, así que sale casi vacía. La startup-config es la configuración guardada completa; por eso el administrador hace antes write memory.

### 3.3 Comprobación y extracción (desde local)

ssh operador@10.10.10.2 "show privilege"
ssh operador@10.10.10.2 "show startup-config" > cisco.cfg


El primer comando devuelve current privilege level is 5. 
El segundo deja la configuración en cisco.cfg.

### 3.4 Alternativa: por FTP (he realizado esta opción puesto que la versión del router que tengo no tiene SSH, añadí la configuración anterior por SSH, porque ahora tengo otro router Cisco que si lo tiene)

configure terminal
privilege exec level 5 copy startup-config
file privilege 5
end
write memory

copy startup-config ftp://jose:CONTRASEÑA@10.10.10.50/cisco.cfg

![alt text](<Captura desde 2026-10-01 22-33-40.png>)


## 4. MikroTik (10.10.10.3)

### 4.1 Limitación de los permisos

En RouterOS los permisos se asignan a grupos mediante políticas. Se creó un grupo lectura con solo dos políticas: ssh (poder entrar por SSH) y read (poder ver).

### 4.2 Configuración como administrador

/user group add name=lectura policy=ssh,read
/user add name=operador group=lectura password=Oper1234
/user group print
/user print

En MikroTik los cambios se aplican y guardan al momento.

### 4.3 Extracción (desde local)

ssh operador@10.10.10.3 "/export" > mikrotik.rsc

![alt text](<Captura desde 2026-10-02 12-31-29.png>)


- /export genera la configuración como script de comandos, que se puede volver a importar con /import.
- Se extrae por la salida de SSH, sin guardar ningún fichero en el router.

## 5. VyOS (10.10.10.1)

### 5.1 Limitación de los permisos

- Existe el nivel de usuario operator, que puede consultar pero no configurar.

### 5.2 Configurar router VyOS como nivel operator

configure
set system login user operador authentication plaintext-password Oper1234
set system login user operador level operator
commit
save


### Extracción desde Ubuntu:

![alt text](<Captura desde 2026-10-01 22-29-51.png>)


## 6. FRRouting (10.10.10.4)

### 6.1 Limitación de los permisos

El usuario operador no tiene acceso a vtysh; es un usuario Linux normal que solo puede leer el fichero /etc/frr/frr.conf.

### 6.2 Requisitos previos como root, en la consola del FRR

SSH instalado y la configuración de FRR guardada en un único fichero:

apk add openssh
ssh-keygen -A
/usr/sbin/sshd
echo "service integrated-vtysh-config" > /etc/frr/vtysh.conf
vtysh -c "write memory"
ls -l /etc/frr/frr.conf

### 6.3 Configuración del operador como root

adduser -D operador
echo "operador:Oper1234" | chpasswd
chmod 755 /etc/frr
chmod 644 /etc/frr/frr.conf


### 6.5 Extracción (desde local)

ssh operador@10.10.10.4 "cat /etc/frr/frr.conf" > frr.conf

![alt text](<Captura desde 2026-10-02 12-39-19.png>)
