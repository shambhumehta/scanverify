# ScanVerify

Spring Boot 4.0.6 + Thymeleaf app that compares an **order/invoice CSV** against a **received-scan CSV** and reports discrepancies.

## Requirements
- Java 17+ (compatible up to Java 26)
- Maven 3.9+

## Run
```bash
mvn spring-boot:run
```
Then open http://localhost:8080

## Login
Hardcoded credentials (in-memory):
- **Username:** `admin`
- **Password:** `admin`

## CSV Formats

**1. Order file** — `barcode,quantity` (header optional) — what was ordered

**2. Received Scan file** — header with `Barcode`, `Quantity`/`Qty`, `Scanner` — units received in GOOD condition
```
Location,Barcode,Qty,Scanner
A1,1001,5,alice
```

**3. Damage file** — header with `Barcode`, `Quantity`/`Qty`, `Scanner` — units received DAMAGED
```
Barcode,Qty,Scanner
1002,1,dave
```

**4. Invoice file** — header with `Barcode` and `Quantity`/`Qty` — what was billed

`Scanner` also accepts `Scanner Name` / `Scanned By` (case-insensitive). Extra columns are ignored in all headered files.

## Report — Four-Way Reconciliation

Accounting model: **good (scanned) + damaged = received**, which should equal **ordered**. Every barcode in any file becomes one row comparing Ordered / Good / Damaged / Received / Invoiced, with status chips:

| Status | Meaning |
|---|---|
| All match | good + damaged = ordered, and invoiced = ordered |
| Shortage | good + damaged < ordered |
| Over-received | good + damaged > ordered |
| Damaged units present | some received units logged as damaged |
| Invoice qty differs | invoiced ≠ ordered |
| Billed for damaged goods | invoiced exceeds good units while damage exists |
| Not scanned / Not invoiced | ordered but absent from that file |
| Not in order / Unexpected barcode | present downstream but never ordered |

A barcode can carry several chips at once. The scanner name (from scan and damage files) is shown per row. Defaults to discrepancies-only with Matched-only / Show-all toggles; Export to Excel honours the selected filter.

Sample files: `sample-orders.csv`, `sample-scans.csv`, `sample-damage.csv`, `sample-invoice.csv`.

## Export to Excel

On the report page, click **Export to Excel** to download an `.xlsx` workbook with two sheets: a **Summary** (counts and totals) and a **Reconciliation** sheet (one row per barcode with Ordered/Scanned/Invoiced, deltas, and status; problem rows shaded, header frozen with autofilter). Generated server-side with Apache POI from the report held in your session.

## Build a jar
```bash
mvn clean package
java -jar target/scanverify-1.0.0.jar
```

## Build a GraalVM Native Image

The `native` Maven profile is configured with GraalVM Native Build Tools.

### Option A — native executable on the host
Requires a **GraalVM JDK** installed with `native-image` on the `PATH`
(easiest via SDKMAN: `sdk install java 21.0.4-graal`).
```bash
mvn -Pnative native:compile
./target/scanverify
```
The app starts in tens of milliseconds. First build takes a few minutes
(GraalVM performs full static analysis of the classpath).

### Option B — native container image (no local GraalVM)
Requires **Docker**. Paketo buildpacks supply GraalVM during the build.
```bash
mvn -Pnative spring-boot:build-image
docker run --rm -p 8080:8080 scanverify:1.0.0
```

### Run the test suite in native mode
```bash
mvn -Pnative test
```

> Notes for native builds: `@Configuration(proxyBeanMethods = false)` is used to
> avoid CGLIB proxies. Spring Boot's AOT processing auto-registers beans,
> controllers and Thymeleaf templates via RuntimeHints, so no manual reflection
> config is needed for this app. The `native-maven-plugin` version is managed by
> the Spring Boot parent BOM.
