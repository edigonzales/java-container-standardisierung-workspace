# P0-Entscheidungsvorschlag · Java-Container für OpenShift

## 1. Executive Summary

**UBI9 als gemeinsame Linux-Basis beschliessen.** Red Hat UBI9 OpenJDK **25** ist der bevorzugte JVM-Standard. Dieselbe Dockerfile-Struktur mit **21** und **17** deckt die nachgewiesenen älteren Artefakte ab. Für beibehaltene Native-Images gilt **UBI9 Minimal**, jeweils nach ABI-, Bibliotheks- und Worker-Prüfung. Damit ergeben sich **vier gemeinsame Base-Image-Varianten und zwei Referenz-Dockerfiles**. Vier weitere Basisreferenzen betreffen den vorgeschlagenen Übergang: Temurin 8, Jenkins LTS, die Java-24-AOT-Basis für ModelFinder und UBI8 für die bisherige Standortkarte. Die effektive Anzahl in OpenShift hängt auch von den Alt-Tags ab. Java 11 für ÖREB ist eine Rückfalloption.

Datenportal Sodata verwendet die Zielbasis bereits in beiden Varianten. Für den ersten Wechsel eines bestehenden Images eignen sich **sodata-stac (Java 21)** und **gretl-job-starter (UBI9 Minimal)**. **sodata-api (Java 17)** dient als einfacher JVM-Vergleich oder Ersatzpilot. Bei ÖREB erlaubt das veröffentlichte Boot-2.7.1-JAR Java 17 gemäss Hersteller. Die Freigabe hängt von PDF-/Font- und XML-Tests ab. Java 11 bleibt dort die Rückfalloption. ModelFinder AOT bleibt bis zur Freigabe des Java-25-Builds auf Java 24, ili2gpkg auf Java 8 und Jenkins auf der offiziellen Jenkins-Basis.

## 2. Wichtigste Feststellungen

- Öffentlich publizierte Images und das zuerst genannte Quellrepo stimmen bei Standortkarte, ilivalidator und ModelFinder nicht überein. Die aktuellen Pfade sind sogis/gwt-standortkarte, sogis/ilivalidatorws und edigonzales/modelfinder mit identischem Codeberg-Stand. Standortkarte ist im aktuellen Registry-Tag Native auf UBI8. Der ältere Nginx-Pfad bleibt als historische Option dokumentiert.
- Datahub und ilivalidatorws publizieren geschichtete Boot-JARs. Beide können später aus demselben ausführbaren Build-Artefakt betrieben werden. Die EXPOSE-Angaben der älteren Datahub-UBI-Variante sind kein Vertrag des aktuell publizierten Images.
- Mehrere alte Tags sind noch abrufbar. STAC v1 benötigt GraalPy. Bei ModelFinder ist gemäss Betriebsangabe ausschliesslich JVM AOT relevant. Für dieses Image sind der konkrete laufende Digest und das JDK-gebundene Startarchiv zu erfassen.
- Bei fünf amd64-Native-Binaries sind glibc-Symbole bis 2.34 und libz direkt belegt. Shared Objects, CA und CPU bleiben Freigabepunkte. Für Standortkarte soll der nächste Build das Native Binary direkt in UBI9 Minimal kopieren. Die UPX-Stufe entfällt. Loader und Bibliotheken lassen sich am unverpackten Artefakt vor dem Test auf dem Worker prüfen.
- Die Registry- und Quellbelege reichen für die nachfolgenden Kandidatenentscheidungen. OpenShift-Pods, effektive Flags und Speichergrenzen, SCCs und Worker-Features sind ohne Clusterkontext offen. [Bestandsaufnahme](P0-inventar.md)

## 3. Empfohlene Runtime-Standards

### Verifizierte Basisimages

Geprüft per Registry-Manifest und amd64-OCI-Config am 18. November 2026. Die Digests bezeichnen den beobachteten Tag-Stand. Die Beispiel-Dockerfiles ergänzen in P2 den zu diesem Zeitpunkt freigegebenen OCI-Index-Digest.

| Kandidat (Tag) | Registry-Digest | Plattformen · Build-Config amd64 | P0-Entscheid |
| --- | --- | --- | --- |
| `registry.access.redhat.com/ubi9/openjdk-25-runtime:latest` | `sha256:7f2eaba33484d04e5331135e8d3e621c5a4740b6b03bc99d25e4d4dc043f6b8c` | amd64, arm64, ppc64le, s390x · 2026-11-10 | Standard JVM 25 |
| `registry.access.redhat.com/ubi9/openjdk-21-runtime:latest` | `sha256:9a074c2ff90be60b12521ffd5e531acafd25fe36f1c177180cbe1176dfb36c84` | amd64, arm64, ppc64le, s390x · 2026-11-10 | Standard JVM 21 |
| `registry.access.redhat.com/ubi9/openjdk-17-runtime:latest` | `sha256:86a55892553aba96b77359af591bf1755d65f414c29c30f081c059f907fbebf7` | amd64, arm64, ppc64le, s390x · 2026-11-10 | Standard JVM 17 |
| `registry.access.redhat.com/ubi9/openjdk-11-runtime:latest` | `sha256:ab65238a8ee25642b4af6c546e96989a7f282e657c190c34e66f7f04c0a8d975` | amd64, arm64, ppc64le, s390x · 2024-12-12 | Verfügbar, öffentlicher Tag alt |
| `registry.access.redhat.com/ubi10/openjdk-25-runtime:latest` | `sha256:dab1be80efb4b8b6e97a314715b367bc9c5ebea3f9c1233b9023910423c45f6c` | amd64, arm64/v8, s390x, ppc64le · 2026-11-11 | Vergleichskandidat, JVM 25 |
| `registry.access.redhat.com/ubi10/openjdk-21-runtime:latest` | `sha256:c1e7e1d56e36b5a88a1c2931bad9c9e5226695238f8b3c814648de10aaf3afc9` | amd64, arm64/v8, s390x, ppc64le · 2026-11-10 | Vergleichskandidat, JVM 21 |
| `registry.access.redhat.com/ubi9/ubi-minimal:latest` | `sha256:baafce2d34783a28b4a472779f304e97e00f9d35c99553bc53b068b14344d9a9` | amd64, arm64/v8, s390x, ppc64le · 2026-11-12 | Standard Native |
| `registry.access.redhat.com/ubi10/ubi-minimal:latest` | `sha256:fb2f7f857cc60d88e3fad19da505083a682ca4569d7b55a61f64aaa43641f741` | amd64, arm64/v8, s390x, ppc64le · 2026-11-12 | Native-Vergleichskandidat |
| `registry.access.redhat.com/ubi9/ubi-micro:latest` | `sha256:dbd325b96bcd432dd42816d31412bdf1dc0fb973b79ed310344bc23c76c068db` | amd64, arm64/v8, s390x, ppc64le · 2026-11-12 | Vergleich, ohne Paketmanager |
| `registry.access.redhat.com/ubi10/ubi-micro:latest` | `sha256:2ad0f98cb13160c4773c945ef93f88b25831c7f27cd3780d72a7e78b210ea72b` | amd64, arm64/v8, s390x, ppc64le · 2026-11-12 | Vergleich, ohne Paketmanager |
| `docker.io/eclipse-temurin:8-jre` | `sha256:1ddd5cc9ac1dac409c938e97db3e39553e89355632cd93ef3bd2604c42a2adc2` | amd64, arm/v7, arm64/v8, ppc64le · 2026-11-16 | Vergleich Java 8, 8u512-b09 auf Ubuntu 26.04 |
| `docker.io/eclipse-temurin:8-jre-noble` | `sha256:7f9f8607f5a8f803fab9f76eb9fa931d1b85c18c77c07ea3548177d905803dc5` | amd64, arm/v7, arm64/v8, ppc64le · 2026-11-16 | Java-8-Ausnahme, 8u512-b09 auf Ubuntu 24.04 |
| `docker.io/eclipse-temurin:11-jre-ubi9-minimal` | `sha256:4ceb696e5ddc86e0e57bdb3f58a1e58cba386128cfc92fbf0582feaefccdbd65` | amd64, arm64/v8, ppc64le, s390x · 2026-11-13 | Java-11-Rückfall, 11.0.33+10 |
| `docker.io/jenkins/jenkins:2.555.2-lts-jdk21` | `sha256:01c992ffef29dcf41c7164e8c16285c657c2368c4943dda8d68c93fdf54447d5` | amd64, arm64, s390x, ppc64le, riscv64 · 2026-05-13 | Jenkins-Ausnahme |
| `docker.io/nginxinc/nginx-unprivileged:stable-alpine` | `sha256:f5116ed4606f3eb015dd4a8e28cb9b351476dc0ad4bc6e9f93b19cec130cdf03` | amd64, arm/v6, arm/v7, arm64, 386, ppc64le, riscv64, s390x · 2026-10-19 | Nginx, falls Altpfad aktiv |
| `registry.access.redhat.com/ubi9/openjdk-8-runtime:latest` | Anfrage HTTP 401 | offen | Frei abrufbarer Runtime-Tag nicht bestätigt |

**UBI9 bevorzugen.** Die direkten Red-Hat-Images für 25/21/17 und UBI9 Minimal waren anonym abrufbar und tragen aktuelle November-2026-Configs. Red Hat dokumentiert die reguläre OpenJDK-Unterstützung bis Ende 2030 (25), 2029 (21) und 2027 (17), mit separaten ELS-Fristen. Der frei abrufbare UBI9-11-Tag existiert, sein beobachteter amd64-Build ist von Dezember 2024. Für die Rückfalloption ÖREB Java 11 eignet sich stattdessen der aktuell gepatchte Temurin 11 auf UBI9 Minimal. Die Java-8-Ausnahme bleibt mit dem geprüften Temurin-Tag 8-jre-noble auf derselben Ubuntu-24.04-Linie wie heute. Der unspezifische 8-jre-Tag zeigte auf Ubuntu 26.04. Ein anonym nutzbarer UBI9 OpenJDK 8 Runtime-Tag wurde nicht bestätigt. [Red Hat OpenJDK Lifecycle](https://access.redhat.com/articles/1299013) · [Adoptium-Support](https://adoptium.net/support/)

UBI10-Images für 25/21 und Minimal sind vorhanden. Der Wechsel bringt für den Bestand keinen erkennbaren Vorteil und erfordert eine eigene Plattformprüfung. RHEL 10 setzt auf x86-64-v3. Ein mit Compatibility-Flag erzeugtes ELF kann CPU-Anforderungen der Betriebssystembasis nicht absenken. UBI Micro ist für den Bestand nachrangig, weil P0 bereits zlib/CA und teilweise tzdata oder weitere Bibliotheken belegt. Paketinstallation ohne eingebauten Paketmanager erhöht den Betriebsaufwand. Beide Minimal-Generationen haben microdnf. Installationen aus den frei verfügbaren UBI-Repositories halten die Build-Abhängigkeiten überschaubar. Zusätzliche RHEL-Entitlements und die Weitergabe von RPMs prüfen, wenn ein benötigtes Paket ausserhalb der UBI-Repos liegt. In P1 die Paketauswahl (insbesondere fontconfig/DejaVu/FreeType) je Architektur verifizieren. [RHEL 10 Plattformanforderungen](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interactively_installing_rhel_over_the_network/system-requirements-and-supported-architectures) · [UBI und Paketquellen](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/assembly_adding-software-to-a-ubi-container_building-running-and-managing-containers)

Die UBI9-Minimal-Basis war lokal auf amd64 und arm64 mit UID 1000830000, GID 0, Read-only-RootFS und ohne Netzwerk startbar. Der arm64-Lauf der OpenJDK-25-Basis unter denselben Bedingungen meldete Red Hat OpenJDK 25.0.5+8 LTS. Das prüft Basis und JRE, der Test mit Anwendungsartefakt und OpenShift-SCC folgt in P2.

### JVM-Kompatibilität des bestehenden Artefakts

Die Bereiche sind Herstellerangaben für den exakten Boot-Release. Sie belegen technische Versionsunterstützung des Frameworks. Die Lauffähigkeit der vollständigen Anwendung mit ihren Dependencies und OpenShift-Werten ist je Pilot zu prüfen. P0 hat keine neuen Java-Versionen per App-Lauf verifiziert. Das Datenportal hat bereits einen Java-25-Publish mit vorgeschalteten CI-Smoke-Tests. Registry-Läufe auf anderen JVM-Versionen belegen nur die entsprechende vorhandene Image-Konfiguration.

| JVM-Artefakt | Heute · Herstellerbereich | Java 25 / 21 ohne Rebuild | P0-Wahl |
| --- | --- | --- | --- |
| Datenportal Sodata 4.1.0 | 25 · Boot 17–26 | 25 bestehender Pfad. 21 unter Bytecode-Ziel. | 25 |
| Datahub 3.3.3 | 21 · Boot 17–22 | 25 oberhalb Herstellergrenze, technisch ungetestet. 21 heute. | 21 |
| STAC 3.1.6 | 17 · Boot 17–21 | 25 oberhalb Herstellergrenze. 21 unterstützt, App-Lauf offen. | 21 |
| Sodata API 3.0.7 | 17 · Boot 17–20 | Beide oberhalb Herstellergrenze. | 17 |
| gb2av 2.7.1 | 17 · Boot 8–18 | Beide oberhalb Herstellergrenze. | 17 |
| Repo-Checker JVM 3.0.2 | 17 · Boot 17–19 | Beide oberhalb Herstellergrenze. | 17 |
| ÖREB Prod / PDF / Branch | 11 · Prod-JAR und PDF/Default-Boot 2.7.1, Boot 8–18. Manueller Branch separat. | Beide oberhalb bestätigter Herstellergrenze. 17 im bestätigten Bereich. | 17 nach PDF-/XML-Abnahme, 11 falls nötig |
| ModelFinder JVM AOT 3.5.3 | 24 · Boot 17–24 | 25 ausserhalb des exakten Releases. 21 unter Java-24-Bytecode. AOT-Archiv zusätzlich JDK-gebunden. | 24 als befristete Ausnahme |
| ilivalidatorws 3.3.4 | 21 · Boot 17–22 | 25 oberhalb Herstellergrenze. 21 heute. | 21 |
| ili2gpkg WS 2.3.3.RELEASE | 8 · Boot 8–14 | Beide oberhalb Herstellergrenze. 11 im Bereich, aber SQLite/JNI und Startcontract wären separat zu testen. | 8 als konservative Wahl |
| Sodata historisch 2.6.7 | 17 · Boot 8–17 | Beide oberhalb Herstellergrenze. | 17, falls der Tag noch läuft |
| STAC v1 3.0.0-RC1 | GraalVM CE 17 mit Python · Boot 17–19 | Java-25/21-Wechsel für dieses Artefakt ohne Polyglot-Nachweis unbelegt. | GraalVM 17 Ausnahme, falls der Tag noch läuft |
| Jenkins LTS 2.555.2 | Controller 21, GRETL-Toolchain 17 | JDK 25 ist für diese Jenkins-LTS-Linie dokumentiert, soweit die Jenkins-eigenen Mindestanforderungen betroffen sind. Der vorliegende Plugin-/Offline-Vertrag bleibt auf 21 + 17. | Offizielles Jenkins-Image |
| Historische Rezepte ohne aktuellen Publisher | Modelfinder GitHub Boot 2.6.7/Java 17. Alter ilivalidator Boot 2.7.1/Java 17. ÖREB-History und ili2gpkg-Prototyp separat im Inventar. | 25 und 21 für die historischen Boot 2.6.7, 2.7.1 und 2.3.1 unbelegt. | Nur bei aktivem Digest auf dessen Runtime abstellen |

Versionsbelege [Boot 2.3.3](https://docs.spring.io/spring-boot/docs/2.3.3.RELEASE/reference/htmlsingle/) · [Boot 2.6.7](https://docs.spring.io/spring-boot/docs/2.6.7/reference/html/getting-started.html) · [Boot 2.7.1](https://docs.spring.io/spring-boot/docs/2.7.1/reference/html/getting-started.html) · [Boot 3.0.0-RC1](https://github.com/spring-projects/spring-boot/blob/v3.0.0-RC1/spring-boot-project/spring-boot-docs/src/docs/asciidoc/getting-started/system-requirements.adoc) · [Boot 3.0.2](https://docs.spring.io/spring-boot/docs/3.0.2/reference/htmlsingle/) · [Boot 3.0.7](https://docs.spring.io/spring-boot/docs/3.0.7/reference/htmlsingle/) · [Boot 3.1.6](https://docs.spring.io/spring-boot/docs/3.1.6/reference/html/getting-started.html) · [Boot 3.2.2](https://github.com/spring-projects/spring-boot/blob/v3.2.2/spring-boot-project/spring-boot-docs/src/docs/asciidoc/getting-started/system-requirements.adoc) · [Boot 3.3.3](https://github.com/spring-projects/spring-boot/blob/v3.3.3/spring-boot-project/spring-boot-docs/src/docs/antora/modules/ROOT/pages/system-requirements.adoc) · [Boot 3.3.4](https://github.com/spring-projects/spring-boot/blob/v3.3.4/spring-boot-project/spring-boot-docs/src/docs/antora/modules/ROOT/pages/system-requirements.adoc) · [Boot 3.5.3](https://github.com/spring-projects/spring-boot/blob/v3.5.3/spring-boot-project/spring-boot-docs/src/docs/antora/modules/ROOT/pages/system-requirements.adoc) · [Boot 3.5.6](https://github.com/spring-projects/spring-boot/blob/v3.5.6/spring-boot-project/spring-boot-docs/src/docs/antora/modules/ROOT/pages/system-requirements.adoc) · [Boot 4.1.0](https://github.com/spring-projects/spring-boot/blob/v4.1.0/documentation/spring-boot-docs/src/docs/antora/modules/ROOT/pages/system-requirements.adoc) · [Jenkins 2.555.2 Systemanforderungen](https://www.jenkins.io/changelog-stable/2.555.2/)

**Spätere Modernisierung.** Für die alten Boot-Releases erfordert offiziell unterstütztes Java 25 einen neu gebauten Release auf einer dann gepflegten Framework-Version, die Java 25 ausdrücklich abdeckt. Der genaue Upgrade-Pfad und nötige Quelländerungen werden erst separat geprüft. Bei ModelFinder wäre mindestens ein Boot-Release mit dokumentiertem Java-25-Support nötig. Boot 3.5.6 belegt, dass bereits ein Patch-Release der 3.5-Linie diese Obergrenze ausweist. Zum späteren Zeitpunkt den gepflegten Patch wählen und vorhandenen Java-24-Bytecode plus neues Release-Artefakt unter 25 testen. Für die tatsächlich eingesetzte AOT-Variante ist app.aot beim Image-Build mit Ziel-JDK 25 pro Zielarchitektur aus dem finalen Anwendungslayout neu zu erzeugen. Start und Indexierung dieses AOT-Images sind die Abnahmekriterien.

## 4. Referenz-Dockerfiles

Die Referenzen sind für bereits erzeugte Artefakte. In vorhandenen Multi-Stage-Projekten wie datenportal-sodata bleiben Build und COPY --from erhalten. Die Beispiele bilden neue Runtime-Stages ab und sind hier nur Entwürfe. UID 1001 ist ein numerischer Fallback ausserhalb OpenShift. Im Pod setzt die SCC die erlaubte UID.

### Ausführbares JVM-JAR

~~~dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-25-runtime:latest
USER 0
WORKDIR /opt/app
COPY --chown=0:0 --chmod=0444 application.jar /opt/app/application.jar
USER 1001:0
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/opt/app/application.jar"]
~~~

Für 21 oder 17 nur den FROM pro Anwendung ändern. Bestehende JVM-Flags, Profile und Ports in der konkreten Variante ergänzen. Quoting und Platzierung der Optionen vor bzw. nach -jar beibehalten. Bei einem bisherigen Hersteller-Launcher, CA-Import oder einer Plattform-Überschreibung zuerst den wirksamen Startvertrag erheben. Alter CMD/ENTRYPOINT und Kubernetes command/args haben unterschiedliche Override-Semantik.

### Native-Binary auf UBI9 Minimal

~~~dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
USER 0
WORKDIR /opt/app
COPY --chown=0:0 --chmod=0555 application /opt/app/application
USER 1001:0
EXPOSE 8080
ENTRYPOINT ["/opt/app/application"]
~~~

Am geprüften UBI9-Minimal-Digest sind zlib, CA-Zertifikate, tzdata, DejaVu Sans und libstdc++ auf amd64 und arm64 bereits enthalten. Deshalb genügt für die Referenz ein COPY. Datenportal übernimmt seine bestehenden Runtime-Pakete und die Zeitzone. Fontconfig und FreeType fehlen im geprüften Grundimage und sind bei tatsächlichem Bedarf gezielt zu ergänzen. Falls ein ELF neuere glibc-Symbole braucht, mit kompatiblem Builder bauen und den Standard erneut prüfen.

### Besondere Artefaktstruktur

Eine dritte gemeinsame Vorlage ist entbehrlich. Die Layer-Extraktion von Datahub, ilivalidatorws und ili2gpkg erfolgt aus ausführbaren JARs. Die risikoärmste erste Migration kann bei diesen Diensten die bestehende Layer-Struktur und den Launcher belassen. Optional danach dasselbe JAR direkt per java -jar starten und Startargumente, Profile, Pfade und Nutzdaten mit gleicher Fixture vergleichen. Das jetzige ModelFinder-AOT-Rezept bleibt eine dokumentierte Ausnahme. Es liefert keinen Grund für eine allgemeine dritte Dockerfile-Vorlage.

STAC und ÖREB behalten im ersten Schritt ihre vollständige Shell-Argumentexpansion mit Quotes und Defaults. Start als exec java aus dem bestehenden Shell-CMD ist ein kleiner, separater Schritt. Falls auf ein Shell-Entrypoint im JSON-Format umgestellt wird und der Pod Argumente anhängt, müssen -c, das erste Argument nach dem Befehl als Shell-$0 und die Weitergabe von "$@" passend abgebildet werden. Keine pauschale Übertragung der Platzhalter in ein Java-Exec-Array. Kein zusätzliches Startskript bei isolierter Runtime-Umstellung.

## 5. Verbindliche OpenShift-Konventionen

- **SCC und Nutzer.** Test unter restricted-v2 bzw. der eingesetzten restriktiven SCC. Laufzeit non-root, keine fest vorgegebene runAsUser im Standard-Pod, kein Privilege Escalation, Capabilities ALL droppen und RuntimeDefault-Seccomp. Gruppenrechte gemäss Plattform mit root group (GID 0) auf den benötigten Pfaden und keine pauschalen Schreibrechte für alle. Für benötigte Pfade beispielsweise install -d -o 0 -g 0 -m 0770 oder gezieltes chgrp 0 und chmod g=u. Numerische Fallback-UID, keine Abhängigkeit von einem benannten Nutzer und kein privilegierter Port. Prüfen, welche Gruppen die Plattform und der konkrete Pod effektiv liefern. [OpenShift Image-Richtlinien](https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html/images/creating-images)
- **Dateisystem.** Artefakte root-owned und nur lesbar/ausführbar. Schreibbares HOME nur für tatsächliche Caches und TMPDIR nur dort ändern, wo die Anwendung es benötigt. Java user.home und java.io.tmpdir unabhängig vom Shell-HOME/TMPDIR prüfen. Für Read-only-RootFS explizite Verzeichnismounts vorsehen, etwa /tmp als emptyDir, plus PVC für dauerhafte Daten. Mounts dürfen vorhandene App-Dateien nicht verdecken. Datenhub /datahub und ilivalidator /ilivalidator enthalten aktuellen Layer-Code. Volume-Rechte separat (Treiber, zugeteilte fsGroup bzw. Plattform-GID) testen. SQLite-Nativbibliothek muss sich bei Bedarf in einen ausführbaren Temp-Pfad extrahieren lassen.
- **CPU-Ziel für Native-Builds.** Bei jedem neuen Build mit aktueller GraalVM Native Image `-march=compatibility` explizit setzen. Abweichende CPU-Ziele erfordern einen dokumentierten Nachweis für alle vorgesehenen Worker. Die CPU-Basis der Distribution und die ABI des Binarys bleiben separate Prüfpunkte. Ein bereits gebautes ELF behält sein CPU-Ziel beim Wechsel des Runtime-Images.
- **Start und Shutdown.** Unprivilegierter Listener üblicherweise 8080. JVM/Binary exec als PID 1, sofern kein erforderlicher Upstream-init wie Jenkins/tini davorsteht. SIGTERM, laufende Uploads/Jobs und passende grace period im Pilot prüfen. Deployment-Overrides, Shell-Expansion, relative Pfade, geerbtes CA-Verhalten und Startreihenfolge beibehalten.
- **Memory.** Limits im Pod mit Lasttest verifizieren. Vorhandene 75/80/90-%-Werte und GC/Profile zuerst erhalten, dabei 90 % unter beengten Limits explizit gegen Threads, Metaspace, Direct Buffers, JNI und cgroups prüfen. Bei neuen Standarddiensten 75 % nur als Mess-Startwert verwenden. Bei Native den Gesamtprozess messen und JVM-Flags nur einsetzen, wenn der Native-Adapter sie wirklich unterstützt. TMP-/Cache-Volumen und Ephemeral-Storage mit berücksichtigen.
- **Probes.** Kubernetes-Probes (Startup bei langer Initialisierung) auf den tatsächlich vorhandenen HTTP-Endpunkten. Readiness darf erforderliche Betriebsabhängigkeiten abbilden. Liveness misst den lokalen Prozesszustand und soll bei kurzzeitig fehlender DB/S3-Verbindung keinen Neustartzyklus auslösen. Zum Start genügt ein überprüfter TCP-Fallback, solange kein verlässlich passender HTTP-Endpunkt existiert. Datenportal trennt /actuator/health/liveness und /actuator/health/readiness bereits. Docker HEALTHCHECK kann für Compose/CI bestehen bleiben.
- **CA, Zeit, Fonts und JNI.** Distro- und Java-Truststore mit derselben öffentlichen und internen CA-Kette testen. Keine vorhandene Laufzeit-Injektion durch Wechsel des Entrypoints verlieren. UTC bzw. Europe/Zurich gemäss Ist-Vertrag, Tests über Sommerzeit. Für ÖREB typografischen Vergleich und Fontconfig/Freetype/DejaVu belegen. Native Library-Auflösung, glibc statt musl, temporäre JNI-Extraktion und Modellcache je Architektur prüfen.

## 6. Ausnahmen und ihre Begründung

| Ausnahme | Entscheidung |
| --- | --- |
| Jenkins | Offizielles Jenkins LTS und Dienst-/Build-JDKs 21/17 beibehalten. Nächsten Jenkins-LTS-Patch gemeinsam mit Plugins, JCasC, Offline-Bundle/DuckDB und PVC/SCC prüfen. |
| ili2gpkg | WebSocket-JVM auf gepatchtem Java 8, numerischer non-root User und Cache-/TMP-Rechte. Die alte Native-Variante entfällt als Ziel. Eine 11er-Option ist später mit demselben JAR und SQLite möglich zu prüfen. |
| ModelFinder JVM AOT | Einzige eingesetzte Variante. Java 24 dient als Ausgangsstand. Für Java 25 einen unterstützenden Boot-Patchstand wählen, app.aot pro Zielarchitektur neu erzeugen und das AOT-Image abnehmen. |
| Standortkarte | Der aktuelle Registry-Stand läuft auf UBI8. Den Digest als Rückfall behalten. Beim ersten Kandidaten-Build das erzeugte Native Binary direkt auf UBI9 Minimal kopieren und UPX samt Kompressions-Stage weglassen. Wenn die alte GraalVM-Toolchain den Build blockiert, Toolchain-Arbeit separat entscheiden. |
| Historische JVM-Tags | Sodata-JVM und STAC-v1-Images vor dem Abbau im Deployment-Inventar suchen. Bei STAC v1 ist GraalPy ein eigener Runtime-Vertrag. Im Bestand aktuell auch Unterschiede zu historischen Modelfinder-/ilivalidator-Publishern beachten. |
| ÖREB Java 11 | Rückfall auf gepflegtes Temurin 11/UBI9, falls Java-17-Fachtests scheitern oder ein abweichender Branch/Tag andere Anforderungen zeigt. |

## 7. Migrationsmatrix je Repository

**A** bedeutet Dockerfile/Basis mit bestehendem Artefakt. **B** bedeutet kleine Build-/CI-Anpassung für die vorgeschlagene Migration. **C** markiert einen zurückgestellten Standardwechsel mit separater Toolchain-/Framework-Arbeit. **S** steht für einen Sonder- oder historischen Pfad, der zunächst betrieblich zugeordnet wird. Alle Kategorien stehen unter den jeweiligen Abnahmekriterien. Die CSV erfasst die Publish-Varianten einzeln. [Maschinenlesbare Matrix](P0-matrix.csv)

| Anwendung | Kategorie und kleinster sinnvoller Schritt |
| --- | --- |
| gretl-job-starter | A · dasselbe amd64-ELF auf UBI9 Minimal, Jenkins/TLS prüfen. |
| gb2av | A · gleicher Release mit Java 17, Modellcache und Export. |
| datenportal-sodata JVM / Native | A · Basen schon im Ziel, bestehende Multi-Stage-Pfade und Smoke-Tests erhalten. |
| datahub | A · Java 21, gleiche Layer oder dasselbe Boot-JAR, Pfade/GC und Fachlauf übernehmen. |
| standortkarte | B · UPX-Stage entfernen und unverpacktes ELF auf UBI9 Minimal prüfen. C nur bei nachgewiesenem Bedarf an einer grösseren Toolchain-Migration. S · historischer Nginx-Pfad. |
| sodata | A · UBI9 Minimal aktualisieren, Index-/GeoJSON-Funktionen prüfen. Historisches JVM-Image S bis Zuordnung. |
| sodata-stac | A · unverändertes JAR auf Java 21, Shell-Vertrag und DB-Test. STAC v1 S. |
| sodata-api | A · unverändertes JAR auf Java 17, /config, Downloads und Temp-Pfade. |
| ÖREB | A · Prod/PDF/Branch auf Java 17 nach releasegenauer PDF-, Karten- und XML-Abnahme. S · JTS-Sondertag zuerst zuordnen und verifizieren. |
| modelfinder | C · ausschliesslich JVM AOT auf Java 25 führen. Boot-Freigabe für 25 und Neuerzeugung von app.aot im Image-Build prüfen. S · die übrigen veröffentlichten Varianten dienen dem Inventar. |
| repo-checker | A · JVM auf Java 17. B · Native auf UBI9 Minimal nach Build-Flag/Builder- bzw. CPU-Ziel-Nachweis. |
| ilivalidatorws | A · Java 21, aktuelle G1-Optionen und SQLite-/Volume-Vertrag. |
| ili2gpkg-web-service-ws | A · Java 8 JRE aktualisieren, Rechte und Startvertrag. Nicht Teil des Native-Standards. |
| datenportal-jenkins-dev | S · offizielle Jenkins-Linie mit bestehenden Integrationsanforderungen. |

## 8. Reihenfolge und Vorbereitung in P1

1. Lesend Deployments, DeploymentConfigs, StatefulSets und CronJobs in den vorgesehenen Namespaces zu Pods, ImageIDs, Architektur, Workload-Owner und registriertem Source-Commit zuordnen. Auch Compose und GitOps-Quellen abgleichen. Metadaten, Pfadnamen, Probes, Ressourcen und Referenzen reichen dafür. Keine Offenlegung von Secret-Werten.
2. Laufende Digests, Artefakt-Hashes und passenden Dockerfile-Pfad festhalten, insbesondere Standortkarte, ÖREB-Tags, ModelFinder AOT, alte STAC/Sodata-Varianten und Jenkins. Den laut Betriebsangabe gemeinsam genutzten OpenShift-Worker-Node, SCC, tatsächliche GID, Mounts und erforderliche UBI-RPMs erfassen. Weitere Worker-CPUs nur einbeziehen, wenn das Scheduling auf diese Nodes möglich ist.
3. Baselines für P2 erfassen. Sicherstellen, dass ein identisches Boot-JAR bzw. ELF verwendet werden kann. Je Pilot festlegen, welche HTTP-, Fach- und Lastprüfungen gegen Baseline und Kandidat laufen. Rollback auf den bisherigen Digest vorbereiten.
4. Nach P2 in kleinen Gruppen vorgehen. Zuerst Runtime-only derselben JVM-Version und der bewusst getestete Wechsel auf 21, dann komplexere Layer/PDF/SQLite, anschliessend Native- und historische Ausnahmen. Separate Freigabe für notwendige Toolchain-Arbeit.

## 9. P2-Piloten und Exit-Kriterien

- **JVM · sodata-stac.** Boot-JAR durch SHA-256 identisch. Neue UBI9 Java 21 Runtime, keine Änderungen am Build. Abnahme mit 17er-Baseline und 21er-Kandidat. Tomcat-, Pool- und Schema-Defaults, Include-Filter, DB-/TLS-Verbindung, ein repräsentativer STAC-Request, Start/SIGTERM, UID, Mounts, Memory unter Last auf beiden vorgesehenen Architekturen.
- **Native · gretl-job-starter.** Dasselbe amd64-ELF von Alpaquita nach UBI9 Minimal. Loader und libz verifizieren, TLS/Jenkins-Trigger mit Testinstanz, Zeitgrenzen unter gleicher Last und beliebiger UID sowie Start auf dem OpenShift-Worker-Node, auf dem die bestehenden Native-Images mit -march=compatibility laufen. Alternativ sodata-api mit bestehendem Java 17 als unabhängiger JVM-Einstieg, falls Native-Worker-Zugriff in P2 noch fehlt.
- **Referenz · datenportal-sodata.** Den vorhandenen Dual-Target-Smoke-Test als Vergleich heranziehen und im vorgesehenen OpenShift-Testnamespace SCC, Snapshot-/Index-Readiness, DNS/HTTPS und SIGTERM prüfen. Keine Neugestaltung des bereits passenden Builders.

## 10. Minimaler CVE- und Update-Prozess

- **OS-Schicht.** Neuer UBI-Patchstand kann Schwachstellen in OS-Paketen und deren Bibliotheken beheben. Für Befunde die Red-Hat-Errata/Backports der konkreten RPMs und Architekturen heranziehen.
- **JVM-Schicht.** Patches innerhalb 25/21/17 und der befristeten Ausnahmen verfolgen. Native bindet Teile der Laufzeit in das Binary ein. Builder-Updates können einen nativen Neubau mit gleicher Anwendungsversion verlangen. Ein neues glibc-Image patcht keine einkompilierten Komponenten oder mitkopierten .so.
- **Java-Dependencies.** Spring Boot, Camel, FOP, Datenbanktreiber und Frameworks müssen separat im Artefakt/SBOM verfolgt werden. Ein Base-Image-Update behebt diese Findings nicht. Die historisch gesetzte Log4j-Variable ist kein Ersatz für eine gepflegte Dependency.
- **Betrieb.** Tag plus Multiarch-Index-Digest pinnen und veröffentlichte Artefakte je Architektur auf ihre Plattform-Digests zurückführen. Renovate-Digest-Updates für vorhandene Dockerfile-Quellen, idealerweise mit einem monatlichen Reviewfenster und einem beschleunigten Weg für Sicherheitsfixes. Bestehende CI bauen und testen lassen, bei externen Artefakten deren Revision/Hash gleich lassen. Rollback-Digest behalten. Bei Rollout alle Manifest-/Plattform-Digests protokollieren. Mit unverändertem Pin gelangt kein Upstream-Patch in den Bestand.
- **P0-Priorität.** Besonders prüfen: alte AdoptOpenJDK-/Liberica-CentOS-/Alpine-Pins, UBI8.3, frühe Quarkus-Micro-Tags, GraalVM-20-/22-Rezepte, Java 24 ohne aktuelle LTS-Pflege, Jenkins-Plugin-/JDK-17-Zweitlaufzeit und floating Alpaquita-Tags. Zuordnung zum tatsächlich laufenden Digest vor der Priorisierung verifizieren. [Red Hat CVE-Daten](https://access.redhat.com/security/security-updates/cve) · [Renovate Docker-Digests](https://docs.renovatebot.com/docker/)

### Scanner-Stichprobe

Docker Scout v1.25.0, 18. November 2026, Registry-Modus, linux/amd64 und Filter critical,high. Angegeben sind die Scanner-Zähler mit genau diesem Filter. Das ist ein Abgleich von Paketversionen mit Advisory-Daten. Er belegt weder Erreichbarkeit im Deployment noch Behebbarkeit, und derselbe Advisory-Eintrag kann mehrere Pakete betreffen.

| Image | Geprüfter Manifest-Index | Geprüftes amd64-Manifest | Scout critical / high |
| --- | --- | --- | --- |
| UBI9 Minimal | `sha256:baafce2d34783a28b4a472779f304e97e00f9d35c99553bc53b068b14344d9a9` | `sha256:27932e0a0da4346c6dffa965960a71ee766117244531635eec25322ecb3e91a6` | 1 / 40 |
| UBI9 OpenJDK 25 Runtime | `sha256:7f2eaba33484d04e5331135e8d3e621c5a4740b6b03bc99d25e4d4dc043f6b8c` | `sha256:f1b7d72c1f7869684210c4a0ac6eeaa7fc5405919d08c23dbdbc896cb25944ab` | 1 / 52 |
| Sodata STAC, publizierter Ist-Stand | `sha256:3bc97ed825ae0c0cda68e0395923bc37d4047e385fd2b23a3bfb4f3b3cc79540` | `sha256:53754e71c2c4bb583cbeca15b621b093d6d6a3cac37de27261d30d80261adc2e` | 14 / 45 |

Bei den UBI9-Basen meldet Scout u. a. Befunde für sqlite-, glibc- und zlib-RPMs. Für STAC zeigt die Stichprobe gleichzeitig Befunde in APK-Paketen (u. a. OpenSSL) und im Boot-JAR (u. a. Tomcat 10.1.16, Spring Web MVC 6.0.14 und PostgreSQL 42.6.0). Der Runtime-Wechsel kann die APK-/JVM-Ebene ersetzen. Die eingebetteten JARs bleiben bestehen. Ein fehlender JVM-Treffer bedeutet keine Aussage über vollständige JVM-Abdeckung. Hersteller-Errata und tatsächliche Nutzung am freigegebenen Digest prüfen. [Docker Scout CVE-CLI](https://docs.docker.com/reference/cli/docker/scout/cves/) · [Red Hat Product Security](https://access.redhat.com/security/)
