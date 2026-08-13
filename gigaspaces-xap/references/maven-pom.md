# XAP 17.3.0 Maven Setup

## Version String

Maven artifact version: **`17.3.0`**

## Repository

GigaSpaces artifacts are NOT on Maven Central. Always add this repository:

```xml
<repositories>
    <repository>
        <id>org.openspaces</id>
        <url>https://maven-repository.openspaces.org</url>
    </repository>
</repositories>
<pluginRepositories>
    <pluginRepository>
        <id>org.openspaces</id>
        <url>https://maven-repository.openspaces.org</url>
    </pluginRepository>
</pluginRepositories>
```

---

## Parent POM Template (multi-module, non-Spring Boot)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-xap-app</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <properties>
        <gigaspaces.version>17.3.0</gigaspaces.version>
        <spring.version>6.2.1</spring.version>
        <slf4j.version>2.0.11</slf4j.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <repositories>
        <repository>
            <id>org.openspaces</id>
            <url>https://maven-repository.openspaces.org</url>
        </repository>
    </repositories>

    <dependencies>
        <!-- Core XAP dependency — pulls in xap-datagrid, xap-common transitively -->
        <dependency>
            <groupId>org.gigaspaces</groupId>
            <artifactId>xap-openspaces</artifactId>
            <version>${gigaspaces.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-orm</artifactId>
            <version>${spring.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>jakarta.annotation</groupId>
            <artifactId>jakarta.annotation-api</artifactId>
            <version>2.1.1</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.6.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Spring Boot Client / REST Application POM

Use this pattern for feeder clients, REST gateways, and standalone apps that connect to a remote space.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-xap-rest-app</artifactId>
    <version>1.0-SNAPSHOT</version>

    <!-- Spring Boot manages Spring dependency versions -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.2</version>
        <relativePath/>
    </parent>

    <properties>
        <gigaspaces.version>17.3.0</gigaspaces.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <repositories>
        <repository>
            <id>org.openspaces</id>
            <url>https://maven-repository.openspaces.org</url>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- XAP core -->
        <dependency>
            <groupId>org.gigaspaces</groupId>
            <artifactId>xap-openspaces</artifactId>
            <version>${gigaspaces.version}</version>
        </dependency>

        <!-- JDBC driver — needed when using JDBC API against the space -->
        <dependency>
            <groupId>com.gigaspaces</groupId>
            <artifactId>xap-jdbc</artifactId>
            <version>${gigaspaces.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Processing Unit (Colocated / Server-Side) POM

Processing Units do NOT use Spring Boot parent; they are packaged as JARs deployed by the GigaSpaces Service Grid.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-processor-pu</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-xap-app</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <dependencies>
        <!-- model module containing @SpaceClass POJOs -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-model</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <build>
        <finalName>my-processor-pu</finalName>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <appendAssemblyId>false</appendAssemblyId>
                    <descriptors>
                        <descriptor>src/main/assembly/assembly.xml</descriptor>
                    </descriptors>
                </configuration>
                <executions>
                    <execution>
                        <id>assembly</id>
                        <phase>package</phase>
                        <goals><goal>single</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Key Artifact Reference

| Artifact | GroupId | When needed |
|----------|---------|-------------|
| `xap-openspaces` | `org.gigaspaces` | Always — core XAP API |
| `xap-datagrid` | `org.gigaspaces` | `com.gigaspaces.*` annotations, SQLQuery, ChangeSet, IdQuery; must be explicit |
| `xap-common` | `org.gigaspaces` | `SmartExternalizable` and other base types; must be explicit |
| `xap-asm` | `org.gigaspaces` | **Spring Boot fat-JAR clients only** — GigaSpaces-repackaged ASM (`org.objectweb.gs.asm.*`) used by `ASMReflectionFactory` at runtime; not on Maven Central — install from `$GS_HOME/lib/required/xap-asm.jar` (see below) |
| `xap-jdbc` | `com.gigaspaces` | JDBC driver / SQL queries via JDBC |
| `xap-spatial` | `org.gigaspaces` | Geospatial queries |
| `xap-full-text-search` | `org.gigaspaces` | Full-text search queries |
| `xap-near-cache` | `com.gigaspaces` | Client-side local cache |
| `xap-security` | `com.gigaspaces` | Secured spaces |
| `xap-mx-rocksdb` | `com.gigaspaces` | MemoryXtend (SSD tier) |

## Local Maven Install (from GigaSpaces installation)

If artifacts are not in the remote repository (air-gapped environments):
```bash
$GS_HOME/bin/gs.sh maven install
```

### Installing xap-asm (required for Spring Boot fat-JAR clients)

`xap-asm` is shipped in `$GS_HOME/lib/required/` but is **not published** to the GigaSpaces Maven repository. Install it once per developer machine:

```bash
mvn install:install-file \
  -Dfile=$GS_HOME/lib/required/xap-asm.jar \
  -DgroupId=org.gigaspaces \
  -DartifactId=xap-asm \
  -Dversion=17.3.0 \
  -Dpackaging=jar
```

Then declare it as a dependency in every Spring Boot client module (feeder, invoice-processor, remoting-client, etc.):

```xml
<dependency>
    <groupId>org.gigaspaces</groupId>
    <artifactId>xap-asm</artifactId>
    <version>${gigaspaces.version}</version>
</dependency>
```

**Symptom when missing:** `java.lang.ClassNotFoundException: org.objectweb.gs.asm.ClassVisitor` thrown during Jini lookup-service discovery at startup.
