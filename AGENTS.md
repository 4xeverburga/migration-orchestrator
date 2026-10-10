# AGENTS — Start the `lotto-sale-desktop` mockup

This folder is **documentation only**. The SP mock library that used to live here now lives in
[`../mock-sp-responses/`](../mock-sp-responses/). This file explains, end to end, how to run the
legacy **`lotto-sale-desktop`** webapp **fully offline** against the mocks.

---

## 0. What "the mockup" is

The legacy WAR talks to two unreachable backends. Both are replaced:

| Backend the app expects | Replaced by | Location | Port |
|---|---|---|---|
| Oracle DB — Stored Procedures via JNDI `DataSource` | **SP mock DataSource** (`pe.com.kyndryl.mock.*`) reading JSON | `mock-sp-responses/` | in-JVM (JNDI) |
| External HTTP services (PlayerWebApi, Bundle, KYC) | **Python HTTP mock** (stdlib only) | `mock-servers/` | `127.0.0.1:8090` |

Reference material (not run): `openapiweb/` (the modernized `ms-*` OpenAPI contracts the mock
shapes are modelled on), `../modernizacion-tinka-pcr21/lotto-sale-desktop/` (the legacy app source).

---

## 1. Prerequisites (absolute paths on this machine)

| Thing | Path | Notes |
|---|---|---|
| **JDK 8** (mandatory) | `/usr/lib/jvm/jdk-8.0.202-oracle-x64` | The app uses `sun.misc.BASE64Encoder`; on JDK 9+ **every URL 404s**. |
| **Apache Tomcat 9** | `/home/ever/tools/apache-tomcat-9.0.83` | `$TC` below. Runs on JDK 8. |
| **Maven** | on `PATH` | Needs JDK 8 for the legacy WAR build. |
| **Python 3** | on `PATH` | Stdlib only — no pip installs needed. |

> ⚠️ **Never inherit the shell `JAVA_HOME`.** The machine default is JDK 21, which breaks the app.
> Always export the JDK 8 path explicitly.

Set up the shell once:

```bash
export JDK8=/usr/lib/jvm/jdk-8.0.202-oracle-x64
export TC=/home/ever/tools/apache-tomcat-9.0.83
export DEMO=/home/ever/Documents/Me/Work/2026/Kyndryl/Tinka/DemoModernizacion
export APP=/home/ever/Documents/Me/Work/2026/Kyndryl/Tinka/modernizacion-tinka-pcr21/lotto-sale-desktop
```

---

## 2. Quick start (copy-paste)

```bash
# --- 1. Build the legacy WAR (JDK 8) ---------------------------------------
cd "$APP"
JAVA_HOME="$JDK8" mvn -B clean package -DskipTests          # -> target/p.war
cp target/p.war "$TC"/webapps/p.war                          # context path = /p

# --- 2. Build + install the SP mock DataSource -----------------------------
cd "$DEMO/mock-sp-responses"
JAVA_HOME="$JDK8" mvn -B clean package -DskipTests          # -> target/sp-mock-datasource-1.0.0.jar
cp target/sp-mock-datasource-1.0.0.jar "$TC"/lib/

# (first time only) declare the JNDI resources — see section 4.

# --- 3. Start the HTTP mock server (leave it running) ----------------------
cd "$DEMO/mock-servers"
python3 server.py &                                          # http://127.0.0.1:8090
python3 configure.py enable --restart                        # point the WAR at it + restart Tomcat

# --- 4. Use the app --------------------------------------------------------
# http://localhost:8080/p/

# --- 5. Tear down / restore real URLs --------------------------------------
python3 configure.py restore --restart
```

JDK-8 jars are already in `$TC/lib/` and `$TC/webapps/p.war` is already deployed, so on a warm
checkout you can normally skip straight to **step 3**.

---

## 3. Steps in detail

### 3.1 Build the legacy WAR
`lotto-sale-desktop` is a Maven project (`<finalName>p</finalName>`, packaging `war`, Java 7
source/target, `ISO-8859-1`). It **only compiles on JDK 8**.

```bash
cd "$APP"
JAVA_HOME="$JDK8" mvn -B clean package -DskipTests
cp target/p.war "$TC"/webapps/p.war
```

Tomcat auto-explodes it to `$TC/webapps/p/`; the context path is `/p`.

### 3.2 Build + install the SP mock DataSource
The Oracle `DataSource` is replaced by `pe.com.kyndryl.mock.MockDataSourceFactory`, which serves
each stored procedure from a JSON file under `mock-sp-responses/`. It is packaged as a shaded jar
(Gson bundled) and dropped into Tomcat's `lib/` so the JNDI factory class is visible to all apps.

```bash
cd "$DEMO/mock-sp-responses"
JAVA_HOME="$JDK8" mvn -B clean package -DskipTests
cp target/sp-mock-datasource-1.0.0.jar "$TC"/lib/
```

The JSON mocks are read **live on every call** — edit `mock-sp-responses/{client,tinka,ticket}/*.json`
and just reload the page, no rebuild/restart.

### 3.3 Point the app at the HTTP mock
`mock-servers/configure.py` rewrites the app's external URLs (they live in
`$JDK8/jre/setup/*.properties`, **outside** the repo) and can restart Tomcat on JDK 8.

```bash
cd "$DEMO/mock-servers"
python3 server.py                 # terminal 1: leave running (127.0.0.1:8090)
python3 configure.py enable --restart
python3 configure.py status       # shows values + "pointed at mock: True|False"
```

It sets `urlPlayerWebApi`/`urlPlayerWebApiLogin`/`urlPlayerWebApiAntiguo` (`playerwebapi.properties`)
and `bundleApiUrl`/`kycApiUrl` (`sale.properties`), backing the originals up as `*.mockservers.bak`.
`restore` puts them back.

### 3.4 Start Tomcat (JDK 8)
`configure.py --restart` already does this, but manually:

```bash
JAVA_HOME="$JDK8" CATALINA_HOME="$TC" CATALINA_BASE="$TC" \
  "$TC"/bin/catalina.sh start
```

> The legacy app logs to a **relative** `D:/LOGS/lotocard.txt`; start `catalina.sh` with
> `cwd="$TC"` so that tree does not pollute the mock folders.

### 3.5 Verify

```bash
curl -so /dev/null -w '%{http_code} %{time_total}s\n' http://localhost:8080/p/
curl -so /dev/null -w '%{http_code} %{time_total}s\n' http://localhost:8080/p/juega-tinka.html
curl -s http://127.0.0.1:8090/health                 # {"status":"UP","version":"1.0.0"}
```

`/p/juega-tinka.html` should answer in **< 1 s**. If it takes ~60 s, the HTTP mock is not being hit
(check `configure.py status` and `$TC/logs/catalina.out`). If everything 404s, Tomcat is on the wrong
JDK.

### 3.6 Regression tests (optional)
The Playwright suite in `test-suite/` drives the app on its own stub (port `8099`). It is a good
smoke test that the mockup is healthy:

```bash
cd "$DEMO/test-suite"
SKIP_ENV_SETUP=1 npm test        # expects 12 passed
```

Note: `configure.py` (8090) and `test-suite/scripts/stub-env.mjs` (8099) each back up what they
replace, so they compose — but **restore in reverse order** and never run two `npm test` invocations
at once (both restart Tomcat → no listener).

> ⚠️ `SKIP_ENV_SETUP=1` keeps the app pointed at **8090** while the suite asserts against its own
> stub expectations. Only use it while `mock-servers` is running.

---

## 4. One-time Tomcat wiring (JNDI)

Tomcat needs the two JNDI resources declared. This machine already has
`$TC/conf/Catalina/localhost/p.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context unpackWAR="true" reloadable="false">
  <Resource name="jdbc/LotoSaleDS"
            auth="Container"
            type="javax.sql.DataSource"
            factory="pe.com.kyndryl.mock.MockDataSourceFactory"
            mockPath="/home/ever/Documents/Me/Work/2026/Kyndryl/Tinka/DemoModernizacion/mock-sp-responses"/>

  <Resource name="jdbc/LottoBdTransDS"
            auth="Container"
            type="javax.sql.DataSource"
            factory="pe.com.kyndryl.mock.MockDataSourceFactory"
            mockPath="/home/ever/Documents/Me/Work/2026/Kyndryl/Tinka/DemoModernizacion/mock-sp-responses"/>
</Context>
```

`mockPath` must be the **`mock-sp-responses` directory** (the JSON files `client/`, `tinka/`,
`ticket/` live there). If you move that folder, update both `mockPath` values here.

Alternative (no Tomcat): run with the JDBC driver directly — class `pe.com.kyndryl.mock.MockDriver`,
URL `jdbc:mock:oracle:thin:@mock`, `-Dmock.sp.path=$DEMO/mock-sp-responses`.

---

## 5. Logs & ports

| What | Where |
|---|---|
| Tomcat app log | `$TC/logs/catalina.out` |
| Mock server log | stdout of `server.py` (e.g. `/tmp/mock.log` if redirected) |
| App | `http://localhost:8080/p/` |
| HTTP mock | `http://127.0.0.1:8090/` (`GET /` lists routes, `GET /health` probes) |
| E2E suite stub | `http://127.0.0.1:8099/` |

`server.py` prints one line per request:
`[MOCK] <ts> POST /getDataBundle scenario=- auth=yes bytes=2 -> …/getDataBundle.json (200)`.

---

## 6. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Every URL returns **404** | Tomcat is running on JDK 9+. Restart on JDK 8. |
| Page hangs ~60 s | HTTP mock unreachable. Run `configure.py enable --restart`; check port 8090. |
| COMPRAR button looks dead | `getPendingDocumentsForApproval` not served → check `mock-servers` log / `configure.py status`. |
| `Mock file not found: …json` in `catalina.out` | `mockPath` in `p.xml` or the JSON file name is wrong. |
| `D:/LOGS/` appears in a folder | Tomcat was started with that folder as `cwd`; start with `cwd="$TC"`. |
| Scenario variant ignored | Name it `<service>.<scenario>.json`; `Path.with_suffix` gotcha — build names with f-strings. |

---

## 7. Related docs

- `../mock-sp-responses/README.md` — the SP mock DataSource (JNDI, driver, supported SPs, tests).
- `../mock-servers/README.md` — the HTTP mock (services, scenarios, editing responses).
- `../test-suite/README.md` — the Playwright regression suite.
