# wb42-sensu
```
apt update && apt install -y curl wget build-essential sudo 
curl -s https://packagecloud.io/install/repositories/sensu/stable/script.deb.sh | bash
curl -s https://packagecloud.io/install/repositories/sensu/community/script.deb.sh | bash
sed -i 's/debian/ubuntu/g' /etc/apt/sources.list.d/sensu_community.list
sed -i 's/buster/bionic/g' /etc/apt/sources.list.d/sensu_community.list
apt update && apt install sensu-go-backend sensu-go-agent sensu-go-cli sensu-plugins-ruby
/opt/sensu-plugins-ruby/embedded/bin/gem install vmstat
mkdir handler && cd handler
wget https://github.com/sensu/sensu-slack-handler/releases/download/1.3.2/sensu-slack-handler_1.3.2_linux_amd64.tar.gz
tar xvf sensu-slack-handler_1.3.2_linux_amd64.tar.gz
cp bin/* /bin && cd .. && rm -r handler
sensu-install -P sensu-plugins-disk-checks,sensu-plugins-http,sensu-plugins-cpu-checks,sensu-plugins-network-checks,sensu-plugins-memory-checks,sensu-plugins-load-checks,sensu-plugins-io-checks
echo "sensu   ALL=(ALL:ALL) NOPASSWD: ALL" >>  /etc/sudoers.d/sensu
chmod 0440 /etc/sudoers.d/sensu
chown root.root /etc/sudoers.d/sensu
```
Creamos el fichero */etc/sensu/backend.yml* con esto:
```
---
state-dir: "/var/lib/sensu/sensu-backend"
agent-port: 18081
api-listen-address: "0.0.0.0:18080" # listen on all IPv4 and IPv6 addresses
log-level: "info"
```
Y reiniciamos:

`systemctl enable --now sensu-backend`

Creamos las variables:
```
export SENSU_BACKEND_CLUSTER_ADMIN_USERNAME=enrique
export SENSU_BACKEND_CLUSTER_ADMIN_PASSWORD=YOUR_PASSWORD
sensu-backend init
sensuctl configure -n --username "$SENSU_BACKEND_CLUSTER_ADMIN_USERNAME" --password "$SENSU_BACKEND_CLUSTER_ADMIN_PASSWORD" --url http://127.0.0.1:18080

service nginx restart
apt install python-cert
certbot --nginx

sensuctl check create check-disks --command '/opt/sensu-plugins-ruby/embedded/bin/check-disk-usage.rb' --interval 300 --subscriptions system
sensuctl check create check-cpu --command '/opt/sensu-plugins-ruby/embedded/bin/check-cpu.rb -w 75 -c 90' --interval 60 --subscriptions system
sensuctl check create check-ram --command '/opt/sensu-plugins-ruby/embedded/bin/check-ram.rb' --interval 60 --subscriptions system
sensuctl check create check-load --command '/opt/sensu-plugins-ruby/embedded/bin/check-load.rb' --interval 60 --subscriptions system
```

Creamos el fichero */etc/sensu/agent.yml*:
```
---
name: "master-sensu01"
subscriptions: 
- system
backend-url:
- "ws://127.0.0.1:18081"
```
Iniciamos el servicio:

`systemctl enable --now sensu-agent`

`sensuctl filter create halfhour --action allow --expressions "event.check.occurrences == 2 || event.check.occurrences % (1800 / event.check.interval) == 0"`

Configuramos las notificaciones por slack
```
sensuctl handler create slack --type pipe --env-vars "SLACK_WEBHOOK_URL= https://hooks.slack.com/services/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" --command "sensu-slack-handler --channel '#alertas'" --filters is_incident,not_silenced,halfhour

sensuctl check update check-disks
```

Vamos a probar a llenar el disco para que salte la alarma:

`dd if=/dev/zero of=file.bin bs=1024M count=34`

Si hacemos re-run de check-disk dos veces manda la alarma a slack
 ![image](https://user-images.githubusercontent.com/65896169/126683665-a9cc65c8-bb51-4ed7-a76d-27d0edb59779.png)


Ahora vamos a probar pushover.

Creamos una aplicación:
 ![image](https://user-images.githubusercontent.com/65896169/126683692-c97131f4-b3c6-4c19-a1e5-875e5023fc60.png)


Ya tenemos creada la aplicación, vamos a crear el fichero para las notificaciones /etc/sensu/handlers/handler-pushover.rbcon esto:

```
#!/usr/bin/env ruby
#
# Sensu Handler: pushover
#
# This handler formats alerts and sends them off to a the pushover.net service.
#
# Released under the same terms as Sensu (the MIT license); see LICENSE
# for details.
require 'net/https'
require 'timeout'
require 'json'
event = JSON.parse(STDIN.read, :symbolize_names => true)
event_name = event[:entity][:metadata][:name] + '/' + event[:check][:metadata][:name]
apiurl = 'https://api.pushover.net/1/messages'
priority = if event[:check][:status] < 3
             event[:check][:status] - 1
           else
             0
           end
params = {
  title: event_name,
  user: 'ug7iewszgyniwkp748h966igwn6mpn',
  token: 'aaf2nzrxhnr99zndpchx3vp25r5sa9',
  priority: priority,
  message: event[:check][:output],
  sound: 'persistent'
}
url = URI.parse(apiurl)
req = Net::HTTP::Post.new(url.path)
res = Net::HTTP.new(url.host, url.port)
res.use_ssl = true
res.verify_mode = OpenSSL::SSL::VERIFY_PEER
begin
  Timeout.timeout(5) do
    req.set_form_data(params)
    res.start { |http| http.request(req) }
    puts "pushover -- sent alert for #{event_name}"
  end
rescue Timeout::Error
  puts "pushover -- timed out while attempting to #{event[:action]} a incident -- #{event_name}"
end
```
Y volvemos a llenar el disco para probar la alerta.

##

Ahora vamos a monitorizar el servicio http y para ello creamos un nuevo check:
```
sensuctl check create sensu.devops-alumnno08.com --command '/opt/sensu-plugins-ruby/embedded/bin/check-http.rb -u https://sensu.devops-alumnno08.com -x BlackDogs -t 45' --interval 60 --subscriptions production-http-checks --handlers pushover,slack
```
Debemos añadirnos a la nueva suscripción que hemos creado (production-http-checks) para poder ver el servicio.

## Monitorizar la máquina de wordpress

Arrancamos la máquina que tenemos wordpress e instalamos sensu:
```
curl -s https://packagecloud.io/install/repositories/sensu/stable/script.deb.sh | bash
curl -s https://packagecloud.io/install/repositories/sensu/community/script.deb.sh | bash
sed -i 's/debian/ubuntu/g' /etc/apt/sources.list.d/sensu_community.list
sed -i 's/buster/bionic/g' /etc/apt/sources.list.d/sensu_community.list
apt update
apt -y install build-essential ruby ruby-dev sensu-go-agent sensu-plugins-ruby
/opt/sensu-plugins-ruby/embedded/bin/gem install vmstat
sensu-install -P sensu-plugins-disk-checks,sensu-plugins-cpu-checks,sensu-plugins-network-checks,sensu-plugins-memory-checks,sensu-plugins-load-checks
echo "sensu   ALL=(ALL:ALL) NOPASSWD: ALL" >>  /etc/sudoers.d/sensu && chmod 0440 /etc/sudoers.d/sensu && chown root.root /etc/sudoers.d/sensu
```
Creamos el fichero */etc/sensu/agent.yml*
```
---
name: "master-web01"
subscriptions:
  - deploy
  - system
  - nginx
  - php-fpm
  - redis
backend-url:
   - "wss://checks.devops-alumno08.com:18082"
```

Y reiniciamos:

`systemctl enable --now sensu-agent.service`

Vamos a monitorizar mysql. Creamos el user:
```
grant all privileges on *.* to 'sensu'@'127.0.0.1' identified by 'unpasswordmuyseguroEnrique';
flush privileges;

apt install default-libmysqlclient-dev
sensu-install -P sensu-plugins-mysql
```
Probamos el servicio en local:
```
/opt/sensu-plugins-ruby/embedded/bin/check-mysql-alive.rb -h 127.0.0.1 -P 3306 -d mysql -u sensu -p unpasswordmuyseguroEnrique
/opt/sensu-plugins-ruby/embedded/bin/check-mysql-connections.rb -h 127.0.0.1 -P 3306 -u sensu -p unpasswordmuyseguroEnrique -a
```
Y creamos los comandos en el servidor de sensu:
```
sensuctl check create mysql-sensu-alive --command '/opt/sensu-plugins-ruby/embedded/bin/check-mysql-alive.rb -h 127.0.0.1 -P 3306 -d mysql -u sensu -p unpasswordmuyseguroEnrique' --interval 60 --subscriptions mysql --handlers slack,pushover
sensuctl check create mysql-sensu-connections --command '/opt/sensu-plugins-ruby/embedded/bin/check-mysql-connections.rb -h 127.0.0.1 -P 3306 -u sensu -p unpasswordmuyseguroEnrique -a' --interval 60 --subscriptions mysql --handlers slack,pushover
```
Acordarse de dar de alta la suscripción a mysql en sensu para ver los checks.

Ahora vamos a comprobar nginx:

En el server de wordpress creamos en el servidor por defecto esta config:
```
location /nginx_status {
	# Enable Nginx stats
	stub_status on;
	# Only allow access from your IP e.g 1.1.1.1 or localhost #
	allow 127.0.0.1;
	# Other request should be denied
	deny all;
}
```
Instalamos el plugin de sensu en el server de wordpress y creamos el comando de check:
```
sensu-install -P sensu-plugins-nginx
sudo /opt/sensu-plugins-ruby/embedded/bin/check-nginx-status.rb -u http://127.0.0.1/nginx_status -w 150
```
Y en el server de sensu creamos el check:

`sensuctl check create nginx-status --command '/opt/sensu-plugins-ruby/embedded/bin/check-nginx-status.rb -u http://127.0.0.1/stub_status -w 150' --interval 60 --handlers slack,pushover --subscriptions nginx`

Y recordar dar de alta en sensu la nueva suscripción que hemos creado.

## Ejercicios
> Pregunta 1
> Investiga como monitorizar el fpm, que seria el útimo servicio que nos faltaría. Cuando lo tengas prepara todo y implementa el check en una nueva subscripción phpfpm. Comprueba que todo funciona adecuadamente

Seguimos la guía de https://github.com/sensu-plugins/sensu-plugins-php-fpm

Buscamos en */etc/php/7.3/fpm/pool.d/www.conf* cual es el path de status:

`pm.status_path = /status`

Creamos en el server de wordpress en la configuración default esta ruta:
```
location /status" {
          include       /etc/nginx/fastcgi_params;
          fastcgi_pass  unix:/var/run/php7.3-fpm.sock;
          fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
          access_log off;
          allow 127.0.0.1;
          deny all;
        }
```
Instalamos el plugin de sensu para php: 

`sensu-install -P sensu-plugins-php-fpm`

Y probamos el comando:

`sudo /opt/sensu-plugins-ruby/embedded/bin/metrics-php-fpm.rb --url http://127.0.0.1/status`

Creamos en el server sensu el check command:

`sensuctl check create php-status --command '/opt/sensu-plugins-ruby/embedded/bin/metrics-php-fpm.rb --url http://127.0.0.1/status' --interval 60 --handlers slack,pushover --subscriptions php-fpm`

Y añadimos la suscripción en la interfaz de sensu

> Pregunta 2
> Instala redis server en la maquina de wordpress. Cuando lo tengas mira como monitorizar el servicio con sensu y crea el check específico dentro de su subscripción redis. Finalmente añade la máquina a esta subcripción y comprueba que todo funciona correctamente.

Instalamos el plugin:

`sensu-install -P sensu-plugins-redis`

Probamos el check

`sudo /opt/sensu-plugins-ruby/embedded/bin/check-redis-ping.rb -h localhost`

Creamos el check en la máquina sensu:

`sensuctl check create redis-status --command '/opt/sensu-plugins-ruby/embedded/bin/check-redis-ping.rb -h localhost' --interval 60 --handlers slack,pushover --subscriptions redis`

> Pregunta 3
> Realiza lo mismo que en la pregunta anterior instalando elastic en la máquina de wordpress. Crea checks para monitorizar el heap space, el cluster health, file descriptors y el estado del nodo.

En nuestra máquina de elastic que tenemos instalamos sensu y configuramos el agente:
Creamos el fichero */etc/sensu/agent.yml*
```
---
name: "elastic-web08"
subscriptions:
  - deploy
  - system
  - elastic
backend-url:
  - "wss://checks.devops-alumno08.com:18082"
```

Y reiniciamos:

`systemctl enable --now sensu-agent.service`

Instalamos el plugin:

`sensu-install -P sensu-plugins-elasticsearch`

Probamos los checks
```
export SSL_CERT_FILE=/etc/filebeat/elasticsearch-ca.pem
/opt/sensu-plugins-ruby/embedded/bin/check-es-node-status.rb -h 95.216.200.57 -u elastic -P elastic -p 9200 --cert /etc/filebeat/elasticsearch-ca.pem -e
/opt/sensu-plugins-ruby/embedded/bin/check-es-file-descriptors.rb  -h 95.216.200.57 -u elastic -P elastic -p 9200 --cert /etc/filebeat/elasticsearch-ca.pem -e
/opt/sensu-plugins-ruby/embedded/bin/check-es-heap.rb  -h 95.216.200.57 -u elastic -W elastic -p 9200 --cert /etc/filebeat/elasticsearch-ca.pem -e -w 2000000000 -c 10000000000
/opt/sensu-plugins-ruby/embedded/bin/check-es-cluster-health.rb  -h 95.216.200.57 -u elastic -P elastic -p 9200
```
Y creamos los comandos:
```
sensuctl check create elastic-status --command '/opt/sensu-plugins-ruby/embedded/bin/check-es-node-status.rb -h 95.216.200.57 -u elastic -P elastic -p 9200 --cert /etc/filebeat/elasticsearch-ca.pem -e' --interval 60 --handlers slack,pushover --subscriptions elastic
sensuctl check create elastic-files --command '/opt/sensu-plugins-ruby/embedded/bin/check-es-file-descriptors.rb  -h 95.216.200.57 -u elastic -P elastic -p 9200 --cert /etc/filebeat/elasticsearch-ca.pem -e' --interval 60 --handlers slack,pushover --subscriptions elastic
sensuctl check create elastic-heap --command '/opt/sensu-plugins-ruby/embedded/bin/check-es-heap.rb  -h 95.216.200.57 -u elastic -W elastic -p 9200 --cert /etc/filebeat/elasticsearch-ca.pem -e -w 2000000000 -c 10000000000' --interval 60 --handlers slack,pushover --subscriptions elastic
sensuctl check create elastic-health --command 'export SSL_CERT_FILE=/etc/filebeat/elasticsearch-ca.pem; /opt/sensu-plugins-ruby/embedded/bin/check-es-cluster-health.rb  -h 95.216.200.57 -u elastic -P elastic -p 9200' --interval 60 --handlers slack,pushover --subscriptions elastic
```

> Pregunta 4
> Levanta el elasticsearch securizado con searchguard e implementa los checks ahí. Recuerdo tener bastantes problemas para hacerlo funcionar así que os dejo los ejemplos que usamos en producción:

```
root@dog-sensu01:~# sensuctl check list | grep elastic
  elasticsearch-cluster-health            sudo bash -c "export SSL_CERT_FILE=/etc/elasticsearch/root-ca.pem ; /opt/sensu-plugins-ruby/embedded/bin/check-es-cluster-health.rb --user checkuser --password supersecurepassword -h `hostname`.blackdogs.io -p 59200"                                                                            60                0     0   elasticsearch-cluster    pushover                                          true       false                                          
  elasticsearch-node-file-descriptors     sudo -u sensu sudo bash -c "export SSL_CERT_FILE=/etc/elasticsearch/root-ca.pem ; /opt/sensu-plugins-ruby/embedded/bin/check-es-file-descriptors.rb --user checkuser --password supersecurepassword -e -h `hostname`.blackdogs.io -p 59200"                                                         60                0     0   elasticsearch            pushover                                          true       false                                          
  elasticsearch-node-heap                 sudo -u sensu sudo bash -c "export SSL_CERT_FILE=/etc/elasticsearch/root-ca.pem ; /opt/sensu-plugins-ruby/embedded/bin/check-es-heap.rb --user checkuser --password supersecurepassword -e -h `hostname`.blackdogs.io -w 80 -c 90 -P -p 59200"                                                      60                0     0   elasticsearch            pushover                                          true       false                                          
  elasticsearch-node-status               sudo -u sensu sudo bash -c "export SSL_CERT_FILE=/etc/elasticsearch/root-ca.pem ; /opt/sensu-plugins-ruby/embedded/bin/check-es-node-status.rb --user checkuser --password supersecurepassword -e -h `hostname`.blackdogs.io -p 59200"                                                              60                0     0   elasticsearch            pushover                                          true       false
  ```
  
> Pregunta 5
> Instala ahora un psql y monitorizalo adecuadamente utilizando sensu.

Instalamos el plugin de postgres:
```
sensu-install -P sensu-plugins-postgres
```
Y creamos el check:
```
sensuctl check create check-postgres --command '/opt/sensu-plugins-ruby/embedded/bin/check-postgres-alive.rb -u root -p mysecurepassword -h localhost -d sensudb' --interval 60 --handlers slack,pushover --subscriptions postgres

```

> Pregunta 6
> Crea ahora un check para monitorizar la caducidad del dominio SSL de wordpress. Idealmente será critical cuando quede menos de un mes para su renovación.

Creamos un script de comprobación:
```
#!/bin/bash
CERTEXPDATE=`date -d "$(openssl x509 -enddate -noout -in /etc/letsencrypt/live/sensu.devops-alumno08.com/fullchain.pem | cut -f2 -d=)"`
ONEMONTHBEF=`date -d "$CERTEXPDATE -1 month" +%s`
AHORA=`date +%s`
if [ $AHORA -gt $ONEMONTHBEF ]; then
	echo "WARNING Queda menos de un mes para que expire el certificado $CERTEXPDATE"
	exit 1
else
	echo "GOOD Fecha de expiración $CERTEXPDATE"
	exit 0
fi
```
Y el comando de check:

`sensuctl check create check-cert --command '/opt/sensu-plugins-ruby/embedded/bin/certexpire.sh' --interval 600 --handlers slack,pushover --subscriptions cert`

> Pregunta 7
> Crea ahora un handler personalizado para recibir las alertas y gestionar resoluciones automaticamente. Importante saber que estamos haciendo a la hora de declarar resoluciones.

Crea un nuevo archivo *tcphandler.json*
```
{
  "type": "Handler",
  "api_version": "core/v2",
  "metadata": {
    "name": "executus",
    "namespace": "default"
  },
  "spec": {
    "type": "tcp",
    "socket": {
      "host": "127.0.0.1",
      "port": 4444
    }
  }
}
```

Luego crea tu recurso de sensu:

`sensuctl create -f tcphandler.json`

Crea un script con tu lenguaje favorito para recibir los eventos:
```
#!/usr/bin/env ruby

require 'socket'
require 'json'

server = TCPServer.open(4444)
loop {
  client = server.accept
  event = JSON.parse(client.gets)
  puts event.inspect
  client.close
}
```
Si enciendes el script, colocas el handler en la alarma de mysql por ejemplo, deberías empezar a recibir eventos.
Una vez lo tengas funcionando, puedes editar el handler en la web para añadir los fitros que usamos normalmente.

Haremos saltar una alarma parando el mysql.

Prueba a hacer que el script solucione el problema lanzando un mysql start cuando reciba el evento de la alarma. Es un escenario de prueba, solo para comprobar que nuestro bot funciona.

Bonus: Cuando haya solucionado la alarma, puedes hacer que haga ssh al host, ejecute de nuevo el check para comprobar si ha solucionado el problema y si no , envíe una alerta critica por otro medio.
