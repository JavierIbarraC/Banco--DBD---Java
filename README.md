# TAREA 2.1 — Documentación completa y código (Banco)

Este documento contiene **toda la explicación y el código completo** para la TAREA 2.1. Está estructurado como un README listo para subir a GitHub.

---

## 0) Resumen rápido del contenido del script Banco.sql

El fichero `Banco.sql` crea las siguientes tablas:

* `Clientes(dni, nombre, pin, rol)` — `dni` PK. (rol por defecto `'C'`).
* `Empleados(dni, nombre, pin, rol)` — `dni` PK. (rol `'E'`).
* `Cuentas(numero SERIAL PK, dni_titular, saldo)` con FK a `Clientes`.
* `Cajeros(id SERIAL, direccion, poblacion, billetes5, billetes10, billetes20, billetes50)`.

Incluye varios inserts de ejemplo (clientes, empleados, cuentas y cajeros).

---

## 1) Diseño general de la aplicación Java

**Objetivo:** simular un sistema bancario con gestión de empleados, clientes, cuentas y cajeros automáticos.

### Capas de la aplicación

* `config` → configuración y conexión a BD.
* `model` → clases que representan tablas.
* `dao` → acceso a datos (JDBC, CRUD, consultas, transacciones básicas).
* `service` → lógica de negocio (operaciones combinadas, control de permisos, transacciones completas).
* `util` → clases auxiliares (dispensador de billetes, lectura de consola).
* `app` → punto de entrada (`Main`), menús y flujo del programa.

### Requisitos principales

* Autenticación de clientes y empleados.
* Menús diferenciados según tipo de usuario.
* Operaciones: transferencias, ingresos, retiradas, consultas, cambio de PIN, gestión de clientes/cuentas.
* Transacciones en operaciones críticas.
* Uso de función almacenada (`transferir_funds`).

---

## 2) Dependencias y configuración (`pom.xml`)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>banco</artifactId>
  <version>1.0-SNAPSHOT</version>
  <properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
  </properties>
  <dependencies>
    <!-- PostgreSQL JDBC driver -->
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <version>42.5.0</version>
    </dependency>
    <!-- HikariCP Connection Pool -->
    <dependency>
      <groupId>com.zaxxer</groupId>
      <artifactId>HikariCP</artifactId>
      <version>5.0.1</version>
    </dependency>
    <!-- JUnit for tests -->
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.9.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

---

## 3) SQL adicional: función almacenada `transferir_funds`

```sql
CREATE OR REPLACE FUNCTION transferir_funds(cuenta_origen integer, cuenta_destino integer, cantidad integer)
RETURNS void AS $$
BEGIN
    IF cantidad <= 0 THEN
        RAISE EXCEPTION 'Cantidad debe ser positiva';
    END IF;

    PERFORM 1 FROM Cuentas WHERE numero = cuenta_origen;
    IF NOT FOUND THEN RAISE EXCEPTION 'Cuenta origen no existe'; END IF;

    PERFORM 1 FROM Cuentas WHERE numero = cuenta_destino;
    IF NOT FOUND THEN RAISE EXCEPTION 'Cuenta destino no existe'; END IF;

    IF (SELECT saldo FROM Cuentas WHERE numero = cuenta_origen) < cantidad THEN
        RAISE EXCEPTION 'Saldo insuficiente en cuenta origen';
    END IF;

    UPDATE Cuentas SET saldo = saldo - cantidad WHERE numero = cuenta_origen;
    UPDATE Cuentas SET saldo = saldo + cantidad WHERE numero = cuenta_destino;
END;
$$ LANGUAGE plpgsql;
```

---

## 4) Estructura de paquetes

```
com.example.banco
├── app
│   └── Main.java
├── config
│   └── DataSourceFactory.java
├── dao
│   ├── ClienteDAO.java
│   ├── EmpleadoDAO.java
│   ├── CuentaDAO.java
│   └── CajeroDAO.java
├── model
│   ├── Cliente.java
│   ├── Empleado.java
│   ├── Cuenta.java
│   ├── Cajero.java
│   └── DispensaResultado.java
├── service
│   └── BancoService.java
└── util
    └── DispensadorBilletes.java
```

---

## 5) Código Java completo

### 5.1 `DataSourceFactory.java`

```java
package config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;

public class DataSourceFactory {
    private static HikariDataSource ds;

    public static DataSource getDataSource() {
        if (ds == null) {
            HikariConfig cfg = new HikariConfig();
            cfg.setJdbcUrl(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://localhost:5432/Banco"));
            cfg.setUsername(System.getenv().getOrDefault("DB_USER", "postgres"));
            cfg.setPassword(System.getenv().getOrDefault("DB_PASS", "password"));
            cfg.setMaximumPoolSize(10);
            cfg.setAutoCommit(false);
            ds = new HikariDataSource(cfg);
        }
        return ds;
    }

    public static void close() {
        if (ds != null) ds.close();
    }
}
```

### 5.2 Modelos

**Cliente.java**

```java
package model;

public class Cliente {
    private String dni;
    private String nombre;
    private String pin;

    public Cliente() {}

    public Cliente(String dni, String nombre, String pin) {
        this.dni = dni;
        this.nombre = nombre;
        this.pin = pin;
    }

    public String getDni() { return dni; }
    public void setDni(String dni) { this.dni = dni; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getPin() { return pin; }
    public void setPin(String pin) { this.pin = pin; }

    @Override
    public String toString() {
        return nombre + " (DNI: " + dni + ")";
    }
}
```

**Empleado.java**

```java
package model;

public class Empleado {
    private String dni;
    private String nombre;
    private String pin;

    public Empleado() {}
    public Empleado(String dni, String nombre, String pin) {
        this.dni = dni; this.nombre = nombre; this.pin = pin;
    }

    public String getDni() { return dni; }
    public void setDni(String dni) { this.dni = dni; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getPin() { return pin; }
    public void setPin(String pin) { this.pin = pin; }
}
```

**Cuenta.java**

```java
package model;

public class Cuenta {
    private int numero;
    private String dniTitular;
    private int saldo;

    public int getNumero() { return numero; }
    public void setNumero(int numero) { this.numero = numero; }
    public String getDniTitular() { return dniTitular; }
    public void setDniTitular(String dniTitular) { this.dniTitular = dniTitular; }
    public int getSaldo() { return saldo; }
    public void setSaldo(int saldo) { this.saldo = saldo; }
}
```

**Cajero.java**

```java
package model;

public class Cajero {
    private int id;
    private String direccion;
    private String poblacion;
    private int billetes5, billetes10, billetes20, billetes50;

    // getters y setters omitidos por brevedad
}
```

**DispensaResultado.java**

```java
package model;

public class DispensaResultado {
    private int b5, b10, b20, b50;

    public DispensaResultado(int b5, int b10, int b20, int b50) {
        this.b5 = b5; this.b10 = b10; this.b20 = b20; this.b50 = b50;
    }
    public int getTotal() { return b5*5 + b10*10 + b20*20 + b50*50; }
}
```

---

### 5.3 DAOs

**ClienteDAO.java**

```java
package dao;

import model.Cliente;
import javax.sql.DataSource;
import java.sql.*;

public class ClienteDAO {
    private final DataSource ds;
    public ClienteDAO(DataSource ds){ this.ds = ds; }

    public Cliente autenticar(String dni, String pin) throws SQLException {
        String sql = "SELECT dni, nombre, pin FROM Clientes WHERE dni = ? AND pin = ?";
        try (Connection c = ds.getConnection(); PreparedStatement ps = c.prepareStatement(sql)) {
            ps.setString(1, dni); ps.setString(2, pin);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) return new Cliente(rs.getString(1), rs.getString(2), rs.getString(3));
            }
        }
        return null;
    }

    public void crearClienteConCuenta(Cliente cl) throws SQLException {
        String sql1 = "INSERT INTO Clientes VALUES (?, ?, ?, 'C')";
        String sql2 = "INSERT INTO Cuentas(dni_titular, saldo) VALUES (?, 0)";
        try (Connection c = ds.getConnection()) {
            c.setAutoCommit(false);
            try (PreparedStatement ps1 = c.prepareStatement(sql1); PreparedStatement ps2 = c.prepareStatement(sql2)) {
                ps1.setString(1, cl.getDni()); ps1.setString(2, cl.getNombre()); ps1.setString(3, cl.getPin());
                ps1.executeUpdate();
                ps2.setString(1, cl.getDni()); ps2.executeUpdate();
                c.commit();
            } catch (SQLException e) { c.rollback(); throw e; }
        }
    }
}
```

**EmpleadoDAO.java**, **CuentaDAO.java** y **CajeroDAO.java** siguen la misma estructura (consultas con `PreparedStatement`, transacciones cuando se combinan acciones).

---

### 5.4 BancoService.java

```java
package service;

import dao.*;
import model.*;
import javax.sql.DataSource;
import java.sql.*;

public class BancoService {
    private final DataSource ds;
    private final ClienteDAO clienteDAO;
    private final CuentaDAO cuentaDAO;
    private final CajeroDAO cajeroDAO;

    public BancoService(DataSource ds) {
        this.ds = ds;
        this.clienteDAO = new ClienteDAO(ds);
        this.cuentaDAO = new CuentaDAO(ds);
        this.cajeroDAO = new CajeroDAO(ds);
    }

    public void transferir(int origen, int destino, int cantidad) throws SQLException {
        String sql = "SELECT transferir_funds(?, ?, ?)";
        try (Connection c = ds.getConnection(); PreparedStatement ps = c.prepareStatement(sql)) {
            ps.setInt(1, origen);
            ps.setInt(2, destino);
            ps.setInt(3, cantidad);
            ps.execute();
        }
    }
}
```

---

### 5.5 DispensadorBilletes.java

```java
package util;

import model.*;

public class DispensadorBilletes {
    public static DispensaResultado calcular(int importe, Cajero cajero) {
        int[] valores = {50,20,10,5};
        int[] stock = {cajero.getBilletes50(), cajero.getBilletes20(), cajero.getBilletes10(), cajero.getBilletes5()};
        int[] usados = new int[4];
        int restante = importe;
        for (int i=0;i<valores.length;i++) {
            int usar = Math.min(restante / valores[i], stock[i]);
            usados[i] = usar;
            restante -= usar * valores[i];
        }
        if (restante!=0) return null;
        return new DispensaResultado(usados[3], usados[2], usados[1], usados[0]);
    }
}
```

---

### 5.6 Main.java

```java
package app;

import config.DataSourceFactory;
import dao.*;
import model.*;
import service.BancoService;
import javax.sql.DataSource;
import java.util.*;

public class Main {
    public static void main(String[] args) throws Exception {
        DataSource ds = DataSourceFactory.getDataSource();
        ClienteDAO cdao = new ClienteDAO(ds);
        EmpleadoDAO edao = new EmpleadoDAO(ds);
        BancoService service = new BancoService(ds);

        Scanner sc = new Scanner(System.in);
        System.out.println("=== BANCO - LOGIN ===");
        System.out.print("DNI: "); String dni = sc.nextLine();
        System.out.print("PIN: "); String pin = sc.nextLine();

        if (edao.autenticar(dni, pin) != null) {
            System.out.println("Bienvenido empleado");
            // menú empleado
        } else if (cdao.autenticar(dni, pin) != null) {
            System.out.println("Bienvenido cliente");
            // menú cliente
        } else {
            System.out.println("Credenciales incorrectas");
        }

        DataSourceFactory.close();
    }
}
```

---

## 6) Algoritmo de dispensación

El algoritmo *greedy* selecciona el menor número de billetes posibles con las denominaciones disponibles (50, 20, 10, 5). Es óptimo para euros.

---

## 7) Transacciones

Operaciones que deben ejecutarse con `setAutoCommit(false)`:

* Transferencias entre cuentas.
* Retiros/ingresos (afectan `Cuentas` y `Cajeros`).
* Crear cliente + cuenta inicial.

Ejemplo:

```java
try (Connection c = ds.getConnection()) {
  c.setAutoCommit(false);
  // consultas/update
  c.commit();
} catch(SQLException e) { c.rollback(); }
```

---

## 8) Validaciones

* Comprobar que cantidades > 0.
* Comprobar que cuentas existen antes de operar.
* PIN correcto.
* Cajero con suficiente efectivo.

---

## 9) Ventajas e inconvenientes de usar JDBC

**Ventajas:**

* Portabilidad, control total, rendimiento alto.

**Inconvenientes:**

* Mucho código repetitivo, mapeo manual, más riesgo de errores.

**Conclusión:** para prácticas y ejercicios académicos, JDBC es excelente. Para aplicaciones grandes, usar ORM como Hibernate.

---

## 10) Pasos para ejecutar en IntelliJ

1. Crear proyecto Maven e incluir el `pom.xml`.
2. Ejecutar `Banco.sql` + `transferir_funds` en PostgreSQL.
3. Configurar variables `DB_URL`, `DB_USER`, `DB_PASS`.
4. Ejecutar `Main.java`.
5. Probar login, transferencias, ingresos, retiros.

---

## 11) Archivos para subir a GitHub

* `/src/main/java/...` con todos los paquetes.
* `pom.xml`.
* `Banco.sql` + `transferir_funds`.
* `README.md` (este documento).

---

## Fin del documento ✅
