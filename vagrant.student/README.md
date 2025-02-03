# Configuración de Servidores DNS con Vagrant y BIND9

Pasos para configurar dos servidores DNS (`atlas` y `ceo`) utilizando Vagrant y BIND9 en un sistema basado en Debian/Ubuntu.

## Pasos

1. **Actualizar los paquetes del sistema:**

    ```bash
    apt-get update
    ```

2. **Instalar Vagrant y VirtualBox:**

    ```bash
    sudo apt-get install -y vagrant virtualbox
    ```

3. **Crear y configurar el archivo Vagrantfile:**

    En el archivo `Vagrantfile` añadimos la configuración global para definir la caja de Vagrant y la memoria:

    ```ruby
    config.vm.box = "debian/bullseye64"
    
    config.vm.provider "virtualbox" do |vb|
        vb.memory = "256"
        vb.linked_clone = true
    end
    ```

    Luego, configuramos el servidor `atlas` con su hostname, dirección IP, instalación de `bind9` y copia de archivos de configuración:

    ```ruby
    config.vm.define "atlas" do |atlas|
        atlas.vm.hostname = "atlas"
        atlas.vm.network "private_network", ip: "192.168.57.2"

        atlas.vm.provision "shell", inline: <<-SHELL
            apt-get update
            apt-get install -y bind9 bind9-utils bind9-doc
            cp /vagrant/resolv.conf /etc/
            cp /vagrant/named.conf.local.atlas /etc/bind/named.conf.local
            cp /vagrant/named.conf.options /etc/bind/named.conf.options
            cp /vagrant/olimpo.test.dns /etc/bind/olimpo.test.dns
            cp /vagrant/192.168.57.dns /etc/bind/192.168.57.dns
            systemctl restart bind9
            systemctl status bind9
        SHELL
    end
    ```

    A continuación, configuramos el servidor `ceo` con su hostname, dirección IP, instalación de `bind9` y copia de archivos de configuración:

    ```ruby
    config.vm.define "ceo" do |ceo|
        ceo.vm.hostname = "ceo"
        ceo.vm.network "private_network", ip: "192.168.57.3"

        ceo.vm.provision "shell", inline: <<-SHELL
            apt-get update
            apt-get install -y bind9 bind9-utils bind9-doc
            cp /vagrant/resolv.conf /etc/
            cp /vagrant/named.conf.local.ceo /etc/bind/named.conf.local
            cp /vagrant/named.conf.options /etc/bind/named.conf.options
            systemctl restart bind9
            systemctl status bind9
        SHELL
    end
    ```

4. **Configurar los archivos `named.conf.local` y `named.conf.options` para atlas:**

    En el archivo `named.conf.local.atlas` añadimos la configuración de la zona directa y la zona inversa:

    ```text
    zone "olimpo.test" {
        type master;
        file "/etc/bind/olimpo.test.dns";
        allow-transfer { 192.168.57.3; };
        notify yes;
    };

    zone "57.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/192.168.57.dns";
        allow-transfer { 192.168.57.3; };
        notify yes;
    };
    ```

    En el archivo `named.conf.options` añadimos la configuración de los reenviadores y otros parámetros:

    ```text
    options {
        directory "/var/cache/bind";
        forwarders { 1.1.1.1; };
        dnssec-validation no;
        listen-on-v6 { any; };
    };
    ```

5. **Configurar los archivos `named.conf.local` y `named.conf.options` para ceo:**

    En el archivo `named.conf.local.ceo` añadimos la configuración para definir `ceo` como servidor secundario:

    ```text
    zone "olimpo.test" {
        type slave;
        file "/var/cache/bind/olimpo.test.dns";
        masters { 192.168.57.2; };
    };

    zone "57.168.192.in-addr.arpa" {
        type slave;
        file "/var/cache/bind/192.168.57.dns";
        masters { 192.168.57.2; };
    };
    ```

6. **Configurar los archivos de zona directa e inversa:**

    En el archivo `olimpo.test.dns` añadimos los registros necesarios para la zona directa:

    ```text
    $TTL    86400
    @       IN      SOA     atlas. hefestos@olimpo.test. (
                            1          ; Serial
                            1200       ; Refresh
                            900        ; Retry
                            2419200    ; Expire
                            604800 )   ; Negative Cache TTL
    ;
    @       IN      NS      atlas
            IN      NS      ceo
    @       IN      A       192.168.57.2
    ceo     IN      A       192.168.57.3
    atenea  IN      A       192.168.57.4
    mercurio IN    A       192.168.57.5
    ares    IN      A       192.168.57.6
    ftp     IN      CNAME   atenea
    smtp    IN      CNAME   mercurio
    pop     IN      CNAME   mercurio
    www     IN      CNAME   ares
    @       IN      MX      10 mercurio
    @       IN      MX      20 dionisio
    ```

    En el archivo `192.168.57.dns` añadimos los registros necesarios para la zona inversa:

    ```text
    $TTL    86400
    @       IN      SOA     atlas. hefestos@olimpo.test. (
                            1          ; Serial
                            1200       ; Refresh
                            900        ; Retry
                            2419200    ; Expire
                            604800 )   ; Negative Cache TTL
    ;
    @       IN      NS      atlas
            IN      NS      ceo
    2       IN      PTR     atlas.olimpo.test.
    3       IN      PTR     ceo.olimpo.test.
    4       IN      PTR     atenea.olimpo.test.
    5       IN      PTR     mercurio.olimpo.test.
    6       IN      PTR     ares.olimpo.test.
    ```

7. **Configurar el archivo `resolv.conf`:**

    En el archivo `resolv.conf` añadimos la configuración del servidor DNS y reenviador:

    ```text
    search olimpo.test
    nameserver 127.0.0.1
    nameserver 1.1.1.1
    ```

8. **Iniciar las máquinas virtuales y verificar el estado de BIND:**

    ```bash
    vagrant up
    vagrant ssh atlas -c "sudo systemctl restart bind9 && sudo systemctl status bind9"
    vagrant ssh ceo -c "sudo systemctl restart bind9 && sudo systemctl status bind9"
    ```

9. **Verificar la configuración de DNS:**

    ```bash
    dig @192.168.57.2 atlas.olimpo.test
    dig @192.168.57.3 ceo.olimpo.test
    dig @192.168.57.2 -x 192.168.57.2
    dig @192.168.57.3 -x 192.168.57.3
    sudo tail -f /var/log/syslog
    ```

## Archivos Incluidos

- `Vagrantfile`
- `named.conf.local.atlas`
- `named.conf.local.ceo`
- `named.conf.options`
- `olimpo.test.dns`
- `192.168.57.dns`
- `resolv.conf`
##


Por: 
- Fernando Bueno Rivera