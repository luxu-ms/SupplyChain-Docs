# SupplyChain dependency inventory

## Executive summary

The three applications use the Spring Boot `2.7.18` parent and Java 8; SupplyChain-Docs has no application dependencies. Backend declares seven direct Maven dependencies, frontend six, and worker four, counting test dependencies. The dependency scopes, build images, and runtime service connections are separate: notably the frontend's provided Tomcat/Jasper artifacts do not establish the externally running Tomcat version.[^b][^f][^w][^d][^fi]

## Confidence assessment

Source manifests and Docker tags are verified at the application pins recorded in `research.md`; the application lexical indexes matched those commits. The Docs lexical index reported `0a26dc4547021e0fb6283a1a467e7fe0101dcc79`, so the Docs tree was checked directly at `f412983392ddd1413fd7ef820b801cf8df1b7792`. The scan covers the four SupplyChain repositories referenced in `research.md`, not unrelated repositories under the account. No resolved dependency tree, advisory scan, image SBOM, or running deployment was obtained.[^d][^b][^f][^w]

## Dependencies

| Repository | Direct Maven dependencies | Container and service dependencies |
| --- | --- | --- |
| [luxu-ms/SupplyChain-Backend](https://github.com/luxu-ms/SupplyChain-Backend) | Web, Data JPA, AMQP, Validation and Test starters `2.7.18`; `mysql:mysql-connector-java:8.0.33` (relocates to `com.mysql:mysql-connector-j`); optional Lombok `1.18.30`. Test starter is test scope.[^b][^bom][^mysql] | `maven:3.8-openjdk-8`, `eclipse-temurin:8-jre`; MySQL `mysql:3306/supplychain`, RabbitMQ `rabbitmq:5672`, and vendor-rating HTTP for the rating endpoint only.[^bi][^bc][^rating] |
| [luxu-ms/SupplyChain-Frontend](https://github.com/luxu-ms/SupplyChain-Frontend) | Web, Tomcat and Test starters `2.7.18`; Jasper `9.0.83`, JSTL `1.2`, optional Lombok `1.18.30`. Tomcat and Jasper are provided; Test starter is test scope.[^f][^bom] | `maven:3.8-openjdk-8`, external `tomcat:8.5-jdk8`; server-side HTTP client targets `http://backend:8080/api` by default.[^fi][^fc] |
| [luxu-ms/SupplyChain-Worker](https://github.com/luxu-ms/SupplyChain-Worker) | Core, AMQP and Test starters `2.7.18`; optional Lombok `1.18.30`. Test starter is test scope.[^w][^bom] | `maven:3.8-openjdk-8`, `eclipse-temurin:8-jre`; RabbitMQ `rabbitmq:5672`, sharing queue names with backend.[^wi][^wc][^bc] |
| [luxu-ms/SupplyChain-Docs](https://github.com/luxu-ms/SupplyChain-Docs) | None. No dependency manifest, lockfile, automation workflow or submodule in the reviewed tree.[^d] | None declared.[^d] |

All three POMs explicitly declare the parent-managed Spring Boot Maven plugin `2.7.18`. Published Maven metadata indicates Spring Framework `5.3.31` across the applications, Jackson `2.13.5` in the web applications, Spring AMQP `2.4.17` and RabbitMQ client `5.14.3` in backend and worker, and Hibernate `5.6.15.Final` plus embedded Tomcat `9.0.83` in backend. These are expected versions, not a resolved runtime inventory.[^b][^f][^w][^bom][^json][^rabbit]

### Recommended baseline for new projects

Use **Spring Boot 4.2.x** with **JDK 21 or 25** for new projects. As of November 26, 2026, **4.2.0** is the latest stable release.[^boot42-release] Keep the chosen JDK consistent in local development, CI, and runtime images. This is a recommendation for new projects, not a change to the Spring Boot 2.7.18 inventory above.

Use the Spring Boot parent or BOM to manage library versions. The **4.2.0** BOM manages the following versions when those dependencies are used:[^boot42-bom]

| Dependency | Managed version |
| --- | --- |
| Spring Framework | `7.1.0` |
| Spring AMQP | `4.2.0` |
| Jackson 3 / Jackson 2 BOMs | `3.1.8` / `2.21.8` |
| Hibernate ORM | `7.4.15.Final` |
| Apache Tomcat | `11.0.27` |
| MySQL Connector/J | `26.10.0` |
| Lombok | `1.18.48` |

### Footnotes

[^d]: [SupplyChain-Docs tree at f412983](https://github.com/luxu-ms/SupplyChain-Docs/tree/f412983392ddd1413fd7ef820b801cf8df1b7792); [research.md:23-30](https://github.com/luxu-ms/SupplyChain-Docs/blob/f412983392ddd1413fd7ef820b801cf8df1b7792/research.md#L23-L30).
[^b]: [SupplyChain-Backend/pom.xml:8-92](https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/pom.xml#L8-L92).
[^f]: [SupplyChain-Frontend/pom.xml:8-87](https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/pom.xml#L8-L87).
[^w]: [SupplyChain-Worker/pom.xml:8-65](https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/pom.xml#L8-L65).
[^bi]: [SupplyChain-Backend/Dockerfile:6-20](https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/Dockerfile#L6-L20).
[^fi]: [SupplyChain-Frontend/Dockerfile:2-22](https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/Dockerfile#L2-L22).
[^wi]: [SupplyChain-Worker/Dockerfile:2-12](https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/Dockerfile#L2-L12).
[^bc]: [SupplyChain-Backend/src/main/resources/application.yml:10-39](https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/resources/application.yml#L10-L39).
[^fc]: [SupplyChain-Frontend/src/main/resources/application.yml:14-18](https://github.com/luxu-ms/SupplyChain-Frontend/blob/c613adbc961e0499d8f2ddb3e087ade7a9a40881/src/main/resources/application.yml#L14-L18).
[^wc]: [SupplyChain-Worker/src/main/resources/application.yml:8-21](https://github.com/luxu-ms/SupplyChain-Worker/blob/458f2bab94b08b3acd167feb04fc485a7ecf6133/src/main/resources/application.yml#L8-L21).
[^rating]: [SupplyChain-Backend/src/main/java/com/acme/scm/service/VendorService.java:36-57](https://github.com/luxu-ms/SupplyChain-Backend/blob/a21f57e84901bad3cf315de2a3520fa8fe34b911/src/main/java/com/acme/scm/service/VendorService.java#L36-L57).
[^bom]: [Published Spring Boot 2.7.18 BOM](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-dependencies/2.7.18/spring-boot-dependencies-2.7.18.pom).
[^mysql]: [MySQL 8.0.33 relocation POM](https://repo.maven.apache.org/maven2/mysql/mysql-connector-java/8.0.33/mysql-connector-java-8.0.33.pom).
[^json]: [Spring Boot JSON starter POM](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-json/2.7.18/spring-boot-starter-json-2.7.18.pom).
[^rabbit]: [Spring Rabbit 2.4.17 POM](https://repo.maven.apache.org/maven2/org/springframework/amqp/spring-rabbit/2.4.17/spring-rabbit-2.4.17.pom).
[^boot42-release]: [Spring Boot 4.2.0 release notes](https://github.com/spring-projects/spring-boot/releases/tag/v4.2.0).
[^boot42-bom]: [Published Spring Boot 4.2.0 BOM](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-dependencies/4.2.0/spring-boot-dependencies-4.2.0.pom).
