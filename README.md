# Java-Container-Standardisierung für OpenShift

Dieser Workspace enthält die P0-Bestandsaufnahme und den Migrationsvorschlag vom 18. November 2026. Die 16 vorgegebenen Quell-Repositories wurden auf ihren Default-Branches untersucht. Neun weitere Checkouts dienten zur Auflösung von Publishern, historischen Varianten und Jenkins-Abhängigkeiten. Quellstände liegen unter `repos/`. Besonders relevant sind die Publisher `sogis/ilivalidatorws`, `edigonzales/modelfinder` und `sogis/gwt-standortkarte`. Für ModelFinder ist gemäss Betriebsangabe ausschliesslich das JVM-AOT-Image migrationsrelevant.

- [Inventar mit Quellbelegen und Registry-Digests](reports/P0-inventar.md)
- [CSV mit 24 Runtime- und Publish-Varianten](reports/P0-matrix.csv)
- [Historischer Entscheidungsvorschlag und Dockerfile-Entwürfe](reports/P0-vorschlag.md)
- [Unabhängiger Review und Abschlussprüfung](reports/P0-review.md)

Die CSV verwendet UTF-8, Kommas als Trennzeichen und eine Zeile je relevantem Pfad. Spiegelregistries und rollende bzw. versionierte Tags derselben Variante sind im Inventar erfasst. Build- und Deployment-Zuordnungen im OpenShift-Cluster bleiben mangels lokalem Clusterkontext Gegenstand von P1.2.

## P1.1 · Verbindliche Runtime-Standards

P1.1 legt UBI9, Java 25/21/17 und die Native-Runtime fest. Für P2 sind `sodata-api` und `gretl-job-starter` als Piloten vorgesehen.

- [Verbindlicher Standard mit Referenz-Dockerfiles](reports/P1.1-standard.md)
- [Unabhängiger Review mit Findings und Abschlussprüfung](reports/P1.1-review.md)
