Proxy Squid Transparente
El Objetivo es Montar un Servidor que sirva para compartir Internet en una red local y además utilizar Squid para hacer un proxy transparente para acelerar la navegación utilizando el puerto 3128, ya que el puerto 80 regularmente esta ocupado por nuestra web.
Antes que nada debo decirte que necesitas dos tarjetas de red, una con salida a internet (en mi ejemplo rl0 con ip 192.168.0.254 y una segunda conectada a la red Local (en mi ejemplo rl1 con ip 192.168.1.254). En la red local debes asignar las ips usando 192.168.1.x netmask 255.255.255.0, puerta de enlace (gateway) 192.168.1.254 o puedes usar un dhcp que te asigne todo esto, además debes utilizar los dns de tu proveedor.

# Configurar NAT

Para hacerlo vamos a compilar un nuevo kernel como vemos a continuación
#cd /usr/src/sys/amd64/conf

#cp GENERIC NUEVO

#ee NUEVO

Incluimos las siguientes opciones para el firewall y el natd.

    options    IPFIREWALL			        # enables IPFW
    options    IPFIREWALL_VERBOSE		    # enables logging for rules with log keyword
    options    IPFIREWALL_VERBOSE_LIMIT=5	# limits number of logged packets per-entry
    options    IPFIREWALL_DEFAULT_TO_ACCEPT # sets default policy to pass what is not explicitly denied
    options    IPDIVERT			            # enables NAT

El paso siguiente es compilar el kernel NUEVO

#config NUEVO
#cd ../../compile/NUEVO

#make depend

#make

#make install

#service ipfw start

#sysctl net.inet.ip.fw.verbose_limit=5

# Comandos IPFW
Lista toda las reglas

#ipfw list

Lista toda las reglas con timestamp

#ipfw -t list

Después de Reiniciar el Servidor, comenzamos la configuración.

# Instalando y Configurando

# Instalar el port del Squid

#pkg install squid

Abre el rc.conf y asegurate que incluya las siguientes líneas
    sysrc  gateway_enable=YES
    sysrc  router_enable=YES
    sysrc  natd_program=/sbin/natd
    sysrc  natd_enable=YES
    sysrc  natd_interface=rl0
    sysrc  firewall_enable=YES
    sysrc  firewall_type=/etc/firewall.rules
    sysrc  firewall_script=/etc/rc.firewall
    sysrc  squid_enable=YES

#ee /etc/rc.conf
    gateway_enable=YES
    router_enable=YES
    natd_program=/sbin/natd
    natd_enable=YES
    natd_interface=rl0
    firewall_enable=YES
    firewall_type=/etc/firewall.rules
    firewall_script=/etc/rc.firewall
    squid_enable=YES

observa que en la línea natd_interface=rl0 se coloca el nombre de la tarjeta de red conectada a internet. en este caso es rl0

# Configurar squid

#cd /usr/local/etc/squid

#cp squid.conf.default squid.conf

#ee squid.conf

Asegurate que existan y están sin comentario las siguientes líneas en el archivo squid.conf (las lineas comentadas comienzan con #)

    cache_mem 48 MB
    maximum_object_size_in_memory 128 KB
    cache_dir ufs /usr/local/squid/cache 200 64 128
    request_body_max_size 4 MB
    acl red src 192.168.0.0/24
    http_access allow red
    cache_mgr tucorreo@tudominio.com
    cache_effective_user squid
    cache_effective_group squid
    visible_hostname tuhost
    httpd_accel_host virtual
    httpd_accel_port 0
    httpd_accel_with_proxy on
    httpd_accel_uses_host_header on
    header_access Via deny all
    header_access X-Forwarded-For deny all

Observa que en cache_mgr debes poner el correo que quieres que aparezca así como en visible_hostname el nombre de tu servidor (Puedes poner el correo y hostname que quieras)

Paso siguiente se inicializa el directorio del cache haciendo lo siguiente:

#squid -z

Terminado el Proceso se ejecuta el squid:

#/usr/loca/etc/rc.d/squid.sh start

# Para terminar creamos el script del firewall

#ee /etc/firewall.rules

    -f flush
    add divert natd all from any to any via rl0
    add fwd 192.168.0.254,3128 tcp from not me to any 80
    add deny log tcp from any yo any in tcpflags syn,fin
    add check-state
    add allow tcp from any to any out keep-state
    add allow all from any to any

En la segunda línea reemplazas rl0 por el nombre de tu tarjeta conectada a internet y en la tercera cambias la ip de la misma tarjeta dejando el puerto 3128 Para terminar reinicias el servidor y debe arrancar automáticamente el demonio squid, puedes revisarlo haciendo esto:
netstat -na

y debe aparecerte la siguiente línea además de otras

    tcp4 0 0 *.3128 *.* LISTEN

si es así felicidades tu proxy esta funcionando para comprobar que funciona navega en algunas paginas en las maquinas de la red (192.168.1.x) y luego checa el archivo access.log:
#ee /usr/local/squid/logs/access.log

y ahí te aparecerá la ip y las paginas que navegaron en cada maquina

# Despedida

Saludos a todos, y si solamente a una persona le sirvió este tutorial,
quedare satisfecho y pensaré que mi trabajo valió la pena...

Hugo Cazares Maldonado
Administrador de Red
Huauchinango, Puebla, Mexico
