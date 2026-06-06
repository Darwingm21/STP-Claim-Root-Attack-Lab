# STP-Claim-Root-Attack-Lab# Ataque STP Claim Root

![Python](https://img.shields.io/badge/Python-3.x-blue)
![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-red)
![Environment](https://img.shields.io/badge/Environment-GNS3%20%7C%20IOSvL2-orange)
![Attack](https://img.shields.io/badge/Attack-STP%20Claim%20Root-purple)
![Mitigation](https://img.shields.io/badge/Mitigation-BPDU%20Guard-darkgreen)
![Use](https://img.shields.io/badge/Use-Controlled%20Lab-yellow)

## Información del proyecto

| Dato                  | Información                                         |
| --------------------- | --------------------------------------------------- |
| Autor                 | Darwing                                             |
| Matrícula             | 2024-2690                                           |
| Docente               | Jonathan Rondon                                     |
| Repositorio           | https://github.com/TuUsuario/STP-Claim-Root-Attack  |
| Video demostrativo    |       https://youtu.be/Cf0_sv3hrOk     |
| Documentación técnica | docs/documentacion-tecnica-profesional.pdf          |
| Red de laboratorio    | 20.24.26.0/24                                       |

---

## Aviso de uso responsable

Este proyecto fue desarrollado únicamente con fines educativos, académicos y de laboratorio controlado. Las pruebas deben ejecutarse solamente en entornos propios o autorizados como GNS3, EVE-NG, PNETLab o laboratorios internos. No debe utilizarse en redes públicas, empresariales o de terceros sin autorización explícita.

---

## Objetivo del laboratorio

Demostrar cómo un atacante conectado a un puerto de acceso puede enviar BPDUs falsas con prioridad 0 para reclamar el rol de **Root Bridge** en una topología STP, alterando el flujo del tráfico de toda la red. Después se aplica **BPDU Guard** en el puerto del atacante para bloquear las BPDUs no autorizadas y proteger la estabilidad de la topología.

---

## Archivos del repositorio

| Archivo                                        | Descripción                                                        |
| ---------------------------------------------- | ------------------------------------------------------------------ |
| `stp_root.py`                                  | Script principal para ejecutar el ataque STP Claim Root.           |
| `mitigacion-stp.md`                            | Documento con las contramedidas aplicadas.                         |
| `README.md`                                    | Guía principal del laboratorio.                                    |
| `docs/documentacion-tecnica-profesional.pdf`   | Documentación técnica profesional detallada del laboratorio.       |
| `images/`                                      | Capturas de pantalla de evidencia del laboratorio.                 |

---

## Topología del laboratorio

La topología usa tres switches en triángulo, ideal para demostrar el impacto en STP.

```
              R1
              |
            [SW-1] ← Root Bridge legítimo (prioridad 32769)
           /       \
       Gi0/0       Gi0/2
      [SW-2]───────[SW-3]
       Gi0/2          Gi0/0
         |
       Kali (atacante)
```

| Dispositivo | Rol                      | Interfaz | Descripción                             |
| ----------- | ------------------------ | -------- | --------------------------------------- |
| SW-1        | Root Bridge legítimo     | Gi0/0~2  | Switch raíz con prioridad 32769         |
| SW-2        | Switch de distribución   | Gi0/0~2  | Switch al que se conecta Kali           |
| SW-3        | Switch de acceso         | Gi0/0~2  | Switch conectado a PC1                  |
| Kali Linux  | Atacante                 | eth0     | Envía BPDUs falsas con prioridad 0      |
| PC1         | Cliente                  | e0       | 20.24.26.92/24                          |

---

## Requisitos previos

- GNS3 con al menos 3 switches Cisco IOSvL2.
- Kali Linux con Python 3 y Scapy instalado.
- Permisos de superusuario (`sudo`).
- Conectividad de capa 2 entre Kali y el switch.

```bash
pip install scapy
```

---

## Parámetros del script

| Parámetro          | Descripción                                               | Ejemplo     |
| ------------------ | --------------------------------------------------------- | ----------- |
| `-i` / `--iface`   | Interfaz de red conectada al switch                       | `-i eth0`   |
| `-m` / `--mac`     | MAC del atacante (default: auto)                          | (opcional)  |
| `-d` / `--delay`   | Intervalo entre BPDUs en segundos (default: 2)            | `-d 2`      |
| `-c` / `--count`   | Cantidad de BPDUs a enviar (0 = infinito)                 | `-c 0`      |
| `-v` / `--verbose` | Muestra cada BPDU enviada                                 | `-v`        |

---

## Flujo del laboratorio

### 1. Estado STP inicial — verificar Root Bridge legítimo

En SW-1, SW-2 y SW-3:

```
SW-1# show spanning-tree vlan 1
SW-2# show spanning-tree vlan 1
SW-3# show spanning-tree vlan 1
```

SW-1 debe aparecer como Root Bridge con prioridad `32769`.

### 2. Ejecutar el ataque desde Kali

```bash
sudo python3 stp_root.py -i eth0 -v
```

El script envía BPDUs con prioridad `0` y MAC `00:00:00:00:00:01`, que es menor que `32769`, forzando a los switches a recalcular el Root Bridge:

```
╔══════════════════════════════════════════════╗
║      STP Root Claim Attack                   ║
║  Interfaz   : eth0                          ║
║  Bridge MAC : 00:00:00:00:00:01             ║
║  Prioridad  : 0 (gana eleccion Root)         ║
╚══════════════════════════════════════════════╝
[   1] BPDU enviado → Root Priority=0  MAC=00:00:00:00:00:01
[   2] BPDU enviado → Root Priority=0  MAC=00:00:00:00:00:01
...
```

Para detener: `Ctrl + C`

### 3. Verificar el cambio del Root Bridge

```
SW-1# show spanning-tree vlan 1
SW-2# show spanning-tree vlan 1
```

El Root Bridge debe haber cambiado a la MAC `0000.0000.0001` con prioridad `0`.

### 4. Aplicar la contramedida — BPDU Guard

En **SW-2** (switch al que está conectada Kali en Gi0/2):

```
enable
configure terminal
spanning-tree vlan 1 priority 4096
interface GigabitEthernet0/2
 description HACIA-KALI-ATACANTE
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
 exit
interface GigabitEthernet0/0
 spanning-tree guard root
 exit
interface GigabitEthernet0/1
 spanning-tree guard root
 exit
end
write memory
```

### 5. Verificar la violación de BPDU Guard

Relanzar el ataque y verificar en SW-2:

```
SW-2# show logging
SW-2# show interfaces status
SW-2# show spanning-tree inconsistentports
```

El puerto Gi0/2 debe quedar en estado **err-disabled** al recibir una BPDU.

### 6. Verificación posterior a la mitigación

```
SW-1# show spanning-tree vlan 1
```

SW-1 debe mantenerse como Root Bridge legítimo aunque el ataque continúe.

---

## Comandos de verificación completos

```
SW-1# show spanning-tree vlan 1
SW-2# show spanning-tree vlan 1
SW-2# show spanning-tree inconsistentports
SW-2# show interfaces GigabitEthernet0/2
SW-2# show logging
SW-2# show interfaces status
```

---

## Video demostrativo

[Ver video del laboratorio en YouTube](https://youtu.be/Cf0_sv3hrOk)

---

## Conclusión

El ataque STP Claim Root demostró que un dispositivo conectado a un puerto de acceso puede manipular la elección del Root Bridge mediante BPDUs falsas con prioridad 0, alterando el flujo del tráfico de toda la topología y permitiendo un posible ataque MitM a nivel de capa 2.

La mitigación con **BPDU Guard** fue efectiva: al recibir una BPDU en el puerto protegido, el switch lo colocó en estado err-disabled automáticamente, bloqueando al atacante y preservando la estabilidad de la topología STP.

---

## Autor

**Darwing**  
Matrícula: **2024-2690**  
Repositorio: https://github.com/TuUsuario/STP-Claim-Root-Attack
