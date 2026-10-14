# Building this fork

Prime's patched fork of [`springml/salesforce-wave-api`](https://github.com/springml/salesforce-wave-api) 1.0.10.
Branch: `feature/oauth-session-support`.

The patch adds **one capability**: build a `PartnerConnection` from a session obtained elsewhere,
instead of always calling the SOAP `login()`. That is what lets Prime ETL authenticate to Salesforce
with OAuth 2.0, and it is required from Salesforce API v65.0 onward, where `login()` no longer
exists.

## What is patched

| File | Change |
|---|---|
| `src/main/java/.../util/SFConfig.java` | a session-based constructor `SFConfig(String sessionId, String serviceEndpoint)`. Built that way, `createPartnerConnection()` calls `setSessionId` / `setServiceEndpoint` on the `ConnectorConfig` instead of `setUsername` / `setPassword`, skipping `login()` entirely. |
| `src/main/java/.../api/APIFactory.java` | `forceAPIWithSession(...)` and `bulkAPIWithSession(...)` — factory methods taking a session instead of credentials. |

`AbstractAPIImpl`, `ForceAPIImpl` and `BulkAPIImpl` are **unchanged**. They only ever ask
`getSfConfig().getPartnerConnection()`, so both the SOQL path and the Bulk API path pick the session
up for free. That is precisely why the patch belongs here rather than further up the stack.

Nothing in this repo knows anything about OAuth — it only knows *"skip the login, here is a
session."* Keep it that way, so a different auth method later needs no change here.

## Build with Java 8

```bash
mvn clean package -DskipTests
# -> target/salesforce-wave-api-1.0.10.jar
```

This compiles on newer JDKs too, but **use Java 8**. The committed jars are bytecode major version
52 (Java 8), and the sibling `spark-salesforce` fork *must* be built on 8 — its sbt 0.13.13 cannot
start on anything newer. Building both on the same JDK keeps the pair consistent.

```bash
sdk install java 8.0.432-tem
sdk use java 8.0.432-tem
unset _JAVA_OPTIONS        # the ETL's --add-opens flags are Java 9+ syntax; Java 8 refuses to start
java -version              # confirm 1.8.x
```

Or in Docker, without installing anything:

```bash
docker run --rm -v "$PWD":/w -w /w -v "$HOME/.m2":/root/.m2 \
  maven:3.8-jdk-8 mvn clean package -DskipTests
```

## Where the jar goes — two places

```bash
# 1. the sibling fork, so it can compile against the patched classes
cp target/salesforce-wave-api-1.0.10.jar ../spark-salesforce/lib/

# 2. the ETL, which vendors this jar directly (N = next qualifier number)
cp target/salesforce-wave-api-1.0.10.jar \
   <etl>/prime-core/lib/salesforce-wave-api-1.0.10-prime-oauth.N.jar
```

`spark-salesforce` keeps its copy as an *unmanaged* jar — deliberately, so Ivy cannot silently
resolve the unpatched 1.0.10 from Maven Central instead.

**After changing anything here you must also rebuild the sibling**, or the ETL gets new wave-api
classes compiled against an old connector. Full steps, including the pom files that have to be kept
in sync:
[`spark-salesforce/BUILD.md`](https://github.com/OdysseyPrime/spark-salesforce/blob/feature/oauth-session-support/BUILD.md)

## Verify

```bash
javap -cp target/salesforce-wave-api-1.0.10.jar \
      com.springml.salesforce.wave.util.SFConfig | grep 'SFConfig(java.lang.String, java.lang.String)'
```

A match means the patch is in the jar. Stock 1.0.10 has no two-argument constructor.

---

Wider context, including how the ETL consumes all of this:
<https://oprime.atlassian.net/wiki/x/AQBHEQE>
