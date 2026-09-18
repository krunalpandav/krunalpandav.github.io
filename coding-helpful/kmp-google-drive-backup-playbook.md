# Google Drive backup in a Kotlin Multiplatform app

A build-it-again guide for Android + iOS, written from a working implementation.

Everything here has shipped and been verified on a real Android device and an iOS simulator. The
code is copied from the working app, not reconstructed, and every "trap" in Part 7 is something
that actually broke during that build — not a list of theoretical risks.

**Read Part 7 before you write any code.** It is the part that saves the days.

---

## Contents

1. [What you are building](#1-what-you-are-building)
2. [The architecture, and why](#2-the-architecture-and-why)
3. [Google Cloud Console setup](#3-google-cloud-console-setup)
4. [Dependencies](#4-dependencies)
5. [The shared code](#5-the-shared-code-commonmain)
6. [The platform code](#6-the-platform-code)
7. [Every trap, and how to avoid it](#7-every-trap-and-how-to-avoid-it)
8. [Test checklist](#8-test-checklist)
9. [Appendix A — encrypting the backup file](#appendix-a--encrypting-the-backup-file)
10. [Appendix B — file association](#appendix-b--file-association)

---

## 1. What you are building

A person taps **Connect**, picks a Google account, and from then on the app keeps one backup file
in a hidden folder inside *their own* Drive. They can back up now, restore, choose a daily or
weekly cadence, or disconnect.

Three rules shape every decision below:

| Rule | Consequence |
|---|---|
| The backup is **their** file, in **their** Drive | You run no server and hold no copy. Nothing to breach, nothing to pay for. |
| The app asks for **one** scope: `drive.appdata` | A per-app hidden folder. You cannot see their documents or photos, and cannot be accused of it. |
| The Drive feature **adds a destination**, not a format | Whatever your app already exports to a file is exactly what goes to Drive. One format, one restore path, one set of bugs. |

If your app does not already have working export/import, build that first. Drive on top of a shaky
export just makes the shaky export harder to debug.

### Why `drive.appdata` and not `drive.file`

`drive.appdata` writes to a folder Drive maintains per application. It is invisible in the person's
Drive UI (they can inspect and delete it under *Settings → Manage apps*), unreadable by any other
app, and — importantly for shipping — **not a restricted scope**, so it needs no Google security
assessment and no CASA audit. `drive.file` and anything broader drag you into review.

---

## 2. The architecture, and why

```
                         commonMain  (all the logic lives here)
   ┌──────────────────────────────────────────────────────────────────┐
   │  DriveBackupRepository ── connect / backupNow / restore /         │
   │          │                disconnect / remoteInfo                 │
   │          ├─ DriveApiClient    → Drive REST over Ktor              │
   │          ├─ DrivePrefs        → DataStore: account, last, cadence │
   │          ├─ AutoBackupRunner  → the one "is it due?" decision     │
   │          └─ DriveStartup      → re-arm + catch-up on foreground   │
   └──────────────────────────────────────────────────────────────────┘
        ▲ interface                ▲ interface            ▲ interface
   GoogleAuthProvider       HttpClientEngine       AutoBackupScheduler
        │                          │                      │
   ┌────┴─────┐              ┌─────┴─────┐          ┌─────┴──────┐
   │ Android  │              │  OkHttp   │          │ WorkManager│
   │ iOS      │              │  Darwin   │          │ BGTask     │
   └──────────┘              └───────────┘          └────────────┘
```

**The platform layer supplies exactly three things.** An HTTP engine, a way to get an access token,
and a way to be woken up on a cadence. Everything else — all the Drive calls, all the state, all
the "should this run" logic — is written once.

Two decisions are worth defending up front, because both are tempting to get wrong:

**Plain REST, not Google's Drive client library.** `com.google.api-client` is JVM-only. Use it and
the whole feature is Android-only. The REST API underneath is six endpoints and works identically
from Ktor on both platforms. This is the single most important structural choice in the document.

**Kotlin never imports Google's iOS SDK.** GoogleSignIn-iOS is a Swift package linked into the app
target. Reaching it from Kotlin/Native via cinterop means vendoring it into the Kotlin build for
three calls. Instead Kotlin declares an `interface GoogleAuthBridge`, Swift implements it, and the
app hands the implementation in at launch.

---

## 3. Google Cloud Console setup

Do this first. Every "it doesn't work" in this feature is 80% likely to be here.

### 3.1 Project and API

1. [console.cloud.google.com](https://console.cloud.google.com) → create a project (or reuse one).
2. **APIs & Services → Library → Google Drive API → Enable.**

Firebase is **not** needed. This is plain OAuth against a Cloud project.

### 3.2 Branding / OAuth consent

**APIs & Services → OAuth consent screen (Branding).**

- **App name** — this is what the person reads on the consent screen. Put the real app name here.
  Not `MyApp - Prod`, not `myapp-android`. They see it.
- User support email, developer contact email, app logo if you have one.
- **User type: External.**

### 3.3 Scope

**Data access → Add or remove scopes → filter for `drive.appdata`.** Add
`https://www.googleapis.com/auth/drive.appdata` and nothing else.

Adding a broader Drive scope "just in case" is how you end up in a security assessment. Don't.

### 3.4 Publishing status

**Audience → Publish app → In production.**

Leave it in *Testing* and two things bite you:

- Only accounts on the test-user list can connect — and iOS enforces this far more strictly than
  Android, because iOS goes through the full browser OAuth flow.
- **Refresh tokens expire after 7 days.** Your beta testers will report "it disconnects itself
  every week" and you will hunt for a bug that isn't in your code.

With only `drive.appdata` requested, publishing to production needs no verification and no review.
It is one button.

### 3.5 OAuth clients

**Credentials → Create credentials → OAuth client ID.** You need **three**, not two:

| # | Type | Fields | When |
|---|---|---|---|
| 1 | Android | package name + **debug** SHA-1 | Now — this is what you develop against |
| 2 | Android | package name + **Play App Signing** SHA-1 | After your first upload to Play |
| 3 | iOS | bundle ID | Now — this one covers everything |

**Android needs two because Android identifies a client by package + signing certificate**, and your
debug builds are signed with a different certificate from your release builds.

Debug SHA-1:

```bash
keytool -list -v -keystore ~/.android/debug.keystore \
        -alias androiddebugkey -storepass android -keypass android | grep SHA1
```

Release SHA-1 comes from **Play Console → Setup → App signing → App signing key certificate**, and
only exists after your first AAB upload. Create client #2 then, or create it now with a placeholder
and edit the SHA-1 later.

**iOS needs only one, ever.** iOS identifies a client by bundle ID alone — there is no certificate
component. One iOS client covers simulator, device, TestFlight and the App Store. If someone tells
you to make an "iOS prod client", they are wrong; it will be byte-for-byte equivalent and will only
confuse you about which ID belongs in `Info.plist`.

You never need the client *secret* for either platform.

### 3.6 What to write down

```
iOS client ID          →  <N>-<hash>.apps.googleusercontent.com
Reversed for the URL   →  com.googleusercontent.apps.<N>-<hash>
```

Both go into `Info.plist`. Android needs nothing in the app at all — Play Services matches your
package name and signature to the client automatically.

---

## 4. Dependencies

`gradle/libs.versions.toml`:

```toml
[versions]
ktor = "3.6.0"
playServicesAuth = "22.0.0"
work = "2.11.2"
coroutines = "1.11.0"
datastore = "1.2.1"

[libraries]
ktor-client-core                = { module = "io.ktor:ktor-client-core",                     version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation",      version.ref = "ktor" }
ktor-serialization-json         = { module = "io.ktor:ktor-serialization-kotlinx-json",      version.ref = "ktor" }
ktor-client-okhttp              = { module = "io.ktor:ktor-client-okhttp",                   version.ref = "ktor" }
ktor-client-darwin              = { module = "io.ktor:ktor-client-darwin",                   version.ref = "ktor" }
play-services-auth              = { module = "com.google.android.gms:play-services-auth",    version.ref = "playServicesAuth" }
coroutines-play-services        = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-play-services", version.ref = "coroutines" }
androidx-work-runtime           = { module = "androidx.work:work-runtime-ktx",               version.ref = "work" }
```

`composeApp/build.gradle.kts`:

```kotlin
commonMain.dependencies {
    implementation(libs.ktor.client.core)
    implementation(libs.ktor.client.content.negotiation)
    implementation(libs.ktor.serialization.json)
}
androidMain.dependencies {
    implementation(libs.ktor.client.okhttp)
    implementation(libs.play.services.auth)
    implementation(libs.coroutines.play.services)   // Task.await()
    implementation(libs.androidx.work.runtime)
}
iosMain.dependencies {
    implementation(libs.ktor.client.darwin)
}
```

iOS also needs the Swift package. In XcodeGen's `project.yml`:

```yaml
packages:
  GoogleSignIn:
    url: https://github.com/google/GoogleSignIn-iOS
    majorVersion: 10.0.0
targets:
  iosApp:
    dependencies:
      - package: GoogleSignIn
        product: GoogleSignIn     # NOT GoogleSignInSwift unless you want their button
```

Declare it in `project.yml`, not by adding it in Xcode's UI — if your `.pbxproj` is generated, a
package added through the UI vanishes the next time anyone runs `xcodegen generate`.

Android manifest:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

iOS `Info.plist`:

```xml
<key>GIDClientID</key>
<string>NNN-hash.apps.googleusercontent.com</string>

<key>CFBundleURLTypes</key>
<array><dict>
  <key>CFBundleURLSchemes</key>
  <array><string>com.googleusercontent.apps.NNN-hash</string></array>
</dict></array>

<!-- only if you want background auto-backup -->
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array><string>com.yourcompany.yourapp.autobackup</string></array>
<key>UIBackgroundModes</key>
<array><string>processing</string></array>
```

---

## 5. The shared code (commonMain)

Ten small files. Create them in this order — each only depends on the ones above it.

### 5.1 `GoogleAuthProvider.kt` — the seam

```kotlin
const val DRIVE_APPDATA_SCOPE = "https://www.googleapis.com/auth/drive.appdata"

/**
 * Where an access token comes from, with the platform left out of it.
 *
 * This is authorization, not sign-in. The app has no accounts and no login screen; connecting an
 * account grants one folder and nothing else.
 */
interface GoogleAuthProvider {
    /** Shows the account picker and consent screen. Only call this from something just tapped. */
    suspend fun connect(): String

    /**
     * A token with no UI at all, for a background job.
     * Throws NeedsConsent rather than prompting: a worker has no screen to prompt on.
     */
    suspend fun accessToken(): String

    /** Drop a token Drive has just rejected, so the next call fetches a fresh one. */
    suspend fun invalidate(token: String)

    /** Forget whatever the platform SDK is holding locally, on disconnect. */
    suspend fun clearSession()
}
```

Four methods. That is the entire platform surface of authorization.

### 5.2 `DriveModels.kt` — the vocabulary

```kotlin
enum class BackupFrequency(val days: Int) { OFF(0), DAILY(1), WEEKLY(7) }

data class DriveAccount(val email: String, val displayName: String?)
data class DriveBackupInfo(val modifiedAtMillis: Long?, val sizeBytes: Long?)

sealed class DriveError(message: String, cause: Throwable? = null) : Exception(message, cause) {
    class NotConnected  : DriveError("No Drive account connected")
    class NeedsConsent  : DriveError("Drive permission must be granted again")
    class Cancelled     : DriveError("Drive permission was cancelled")
    class Network(cause: Throwable? = null) : DriveError("Network error", cause)
    class Auth(val detail: String)          : DriveError("Google auth error: $detail")
    class Http(val code: Int, val body: String) : DriveError("Drive HTTP $code")
    class NoBackupFound : DriveError("No backup in this account")
    class Unexpected(cause: Throwable) : DriveError(cause.message ?: "Unexpected", cause)
}
```

**Make this sealed class rich from day one.** Every one of these has a different message in the UI
and a different retry policy in the background worker. Collapsing them into one `DriveException`
means that when something breaks in six months you get "Something went wrong" and no way to tell a
dead network from a revoked grant.

`Auth` deliberately carries a `detail` string. When the platform SDK fails in a way you did not
anticipate, that string is the only evidence you will have.

### 5.3 `DriveApi.kt` — the six REST calls

```kotlin
const val DRIVE_BACKUP_FILE_NAME = "yourapp-backup.json"

@Serializable
data class DriveFile(
    val id: String,
    val name: String? = null,
    val size: String? = null,          // Drive sends int64 as a STRING
    val modifiedTime: String? = null,  // RFC 3339
)

@Serializable internal data class DriveFileList(val files: List<DriveFile> = emptyList())
@Serializable internal data class DriveCreateMeta(val name: String, val parents: List<String>, val mimeType: String)
@Serializable internal data class DriveAbout(val user: DriveUser)
@Serializable internal data class DriveUser(val displayName: String? = null, val emailAddress: String? = null)

/**
 * A client of its own, not the app's. If your app ever grows its own API client, whatever that
 * carries by default — a base URL, an auth header, an interceptor — must not reach Google.
 */
fun createDriveHttpClient(engine: HttpClientEngine): HttpClient = HttpClient(engine) {
    expectSuccess = false                 // authed() must SEE a 401, not have it thrown at it
    install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
    install(HttpTimeout) {
        connectTimeoutMillis = 15_000
        requestTimeoutMillis = 120_000    // a backup is one upload; let it take a while
    }
}

class DriveApiClient(
    private val http: HttpClient,
    private val auth: GoogleAuthProvider,
) {
    private val json = Json { encodeDefaults = true }

    /**
     * Who the token belongs to.
     * Takes the token directly because connect() has one in hand and has not yet saved the
     * account it belongs to — which is the very thing this call is being made to find out.
     */
    suspend fun about(token: String? = null): DriveAccount {
        val about: DriveAbout = authed(token) { t ->
            http.get("$API/about") { bearerAuth(t); parameter("fields", "user(displayName,emailAddress)") }
        }.body()
        val email = about.user.emailAddress ?: throw DriveError.Auth("Account has no email address")
        return DriveAccount(email, about.user.displayName)
    }

    suspend fun find(name: String): DriveFile? =
        authed { t ->
            http.get("$API/files") {
                bearerAuth(t)
                parameter("spaces", "appDataFolder")
                parameter("q", "name = '$name' and trashed = false")
                parameter("fields", "files($FIELDS)")
                parameter("orderBy", "modifiedTime desc")
                parameter("pageSize", 1)
            }
        }.body<DriveFileList>().files.firstOrNull()

    suspend fun create(name: String, bytes: ByteArray): DriveFile {
        val boundary = "up-" + Random.nextLong().toULong().toString(16)
        val meta = json.encodeToString(
            DriveCreateMeta.serializer(),
            DriveCreateMeta(name, listOf("appDataFolder"), JSON_MIME),
        )
        return authed { t ->
            http.post("$UPLOAD/files") {
                bearerAuth(t)
                parameter("uploadType", "multipart")
                parameter("fields", FIELDS)
                setBody(ByteArrayContent(
                    multipartRelated(boundary, meta, bytes),
                    ContentType.MultiPart.Related.withParameter("boundary", boundary),
                ))
            }
        }.body()
    }

    /** Replaces the contents, so the account keeps exactly one backup. */
    suspend fun update(fileId: String, bytes: ByteArray): DriveFile =
        authed { t ->
            http.patch("$UPLOAD/files/$fileId") {
                bearerAuth(t)
                parameter("uploadType", "media")
                parameter("fields", FIELDS)
                setBody(ByteArrayContent(bytes, ContentType.Application.Json))
            }
        }.body()

    suspend fun download(fileId: String): String =
        authed { t -> http.get("$API/files/$fileId") { bearerAuth(t); parameter("alt", "media") } }
            .bodyAsText()

    /** Best effort: local state is already cleared, and a person disconnecting on a train
     *  should not be told it failed. */
    suspend fun revoke(token: String) {
        runCatching { http.submitForm(REVOKE_URL, parameters { append("token", token) }) }
    }

    private suspend fun authed(
        initialToken: String? = null,
        send: suspend (String) -> HttpResponse,
    ): HttpResponse {
        val token = initialToken ?: auth.accessToken()
        var res = network { send(token) }

        if (res.status == HttpStatusCode.Unauthorized) {
            // A token handed in by connect() is the freshest there will ever be, and the account
            // it belongs to has not been saved yet — so asking for another would fail with
            // "not connected" and bury the real problem under the wrong message.   <-- TRAP 4
            if (initialToken != null) throw DriveError.NeedsConsent()
            auth.invalidate(token)
            res = network { send(auth.accessToken()) }
            if (res.status == HttpStatusCode.Unauthorized) throw DriveError.NeedsConsent()
        }
        if (!res.status.isSuccess()) {
            val body = res.bodyAsText()
            // The consent screen lets the Drive checkbox be unticked. The token is then real and
            // valid, and simply not allowed to do this — which reads as 403, not 401.
            if (res.status == HttpStatusCode.Forbidden && "insufficient" in body.lowercase()) {
                throw DriveError.NeedsConsent()
            }
            val excerpt = body.take(500)
            println("Drive: HTTP ${res.status.value} — $excerpt")
            throw DriveError.Http(res.status.value, excerpt)
        }
        return res
    }

    /**
     * Anything the engine throws is the network failing. Ktor's engines do not share an exception
     * type across platforms — OkHttp throws IOException, Darwin throws its own — so catch broadly.
     */
    private suspend fun network(block: suspend () -> HttpResponse): HttpResponse =
        try { block() }
        catch (e: DriveError) { throw e }
        catch (e: Exception) {
            println("Drive: network failure — ${e::class.simpleName}: ${e.message}")
            throw DriveError.Network(e)
        }

    private companion object {
        const val API        = "https://www.googleapis.com/drive/v3"
        const val UPLOAD     = "https://www.googleapis.com/upload/drive/v3"
        const val REVOKE_URL = "https://oauth2.googleapis.com/revoke"
        const val FIELDS     = "id,name,size,modifiedTime"
        const val JSON_MIME  = "application/json"
    }
}

/** Drive's multipart upload: the metadata as one part, the file as the next. */
private fun multipartRelated(boundary: String, metadataJson: String, content: ByteArray): ByteArray {
    val head = buildString {
        append("--$boundary\r\n")
        append("Content-Type: application/json; charset=UTF-8\r\n\r\n")
        append(metadataJson)
        append("\r\n--$boundary\r\n")
        append("Content-Type: application/json\r\n\r\n")
    }.encodeToByteArray()
    return head + content + "\r\n--$boundary--\r\n".encodeToByteArray()
}
```

Note `authed()` — the retry-once-then-give-up rule lives in exactly one place. Every call gets it,
and no call gets it twice.

### 5.4 `DrivePrefs.kt` — five keys in DataStore

```kotlin
data class DrivePrefsState(
    val account: DriveAccount? = null,
    val lastBackupMillis: Long? = null,
    val lastSizeBytes: Long? = null,
    val frequency: BackupFrequency = BackupFrequency.OFF,
)

/**
 * Shares the one DataStore the rest of the app uses rather than opening a second: two stores over
 * the same directory is a way to lose writes, and these are five keys.
 */
class DrivePrefs(private val store: DataStore<Preferences>) {

    val state: Flow<DrivePrefsState> = store.data.map { p ->
        DrivePrefsState(
            account = p[EMAIL]?.let { DriveAccount(it, p[NAME]) },
            lastBackupMillis = p[LAST_AT],
            lastSizeBytes = p[LAST_SIZE],
            frequency = BackupFrequency.entries.firstOrNull { it.name == p[FREQ] } ?: BackupFrequency.OFF,
        )
    }

    suspend fun snapshot() = state.first()
    suspend fun connectedEmail() = snapshot().account?.email
    suspend fun frequency() = snapshot().frequency
    suspend fun lastBackupMillis() = snapshot().lastBackupMillis

    suspend fun setAccount(account: DriveAccount) { /* EMAIL, NAME */ }
    suspend fun setLastBackup(atMillis: Long, sizeBytes: Long) { /* LAST_AT, LAST_SIZE */ }
    suspend fun setFrequency(frequency: BackupFrequency) { /* FREQ */ }

    /**
     * Forgets the account and the backup that belonged to it. The cadence is deliberately KEPT:
     * it is a preference about how the person likes to work, and should still be set the way they
     * left it when they connect a different account.
     */
    suspend fun clearAccount() {
        store.edit { it.remove(EMAIL); it.remove(NAME); it.remove(LAST_AT); it.remove(LAST_SIZE) }
    }

    private companion object {
        val EMAIL     = stringPreferencesKey("drive_account_email")
        val NAME      = stringPreferencesKey("drive_account_name")
        val LAST_AT   = longPreferencesKey("drive_last_backup_at")
        val LAST_SIZE = longPreferencesKey("drive_last_backup_size")
        val FREQ      = stringPreferencesKey("drive_backup_frequency")
    }
}
```

### 5.5 `DriveBackupRepository.kt` — the five operations

This is the file to read most carefully. Two comments in it describe rules that protect data.

```kotlin
class DriveBackupRepository(
    private val drive: DriveApiClient,
    private val auth: GoogleAuthProvider,
    private val prefs: DrivePrefs,
    private val backup: BackupRepository,     // YOUR existing export/import
) {
    /**
     * One backup or restore at a time, across every caller.
     *
     * The dangerous pair is an automatic backup starting while a restore is running: the upload
     * would carry the half-replaced database, and overwrite the very file being restored from.
     */
    private val lock = Mutex()

    /**
     * Connects an account, and REPORTS whatever backup is already in it.
     *
     * A backup found here is returned, never acted on: the person may be connecting the account
     * their old phone used, and quietly overwriting it with a new install's empty database is the
     * one unrecoverable thing this feature could do. When the folder is empty there is nothing to
     * lose, so the first backup is taken immediately.
     */
    suspend fun connect(): DriveBackupInfo? {
        val token = auth.connect()
        val account = drive.about(token)
        prefs.setAccount(account)
        val existing = drive.find(DRIVE_BACKUP_FILE_NAME)?.toInfo()
        if (existing == null) backupNow()
        return existing
    }

    suspend fun backupNow(): Unit = lock.withLock {
        val text = backup.exportBackup().getOrElse { throw DriveError.Unexpected(it) }
        val bytes = text.encodeToByteArray()
        val existing = drive.find(DRIVE_BACKUP_FILE_NAME)
        if (existing == null) drive.create(DRIVE_BACKUP_FILE_NAME, bytes)
        else drive.update(existing.id, bytes)
        prefs.setLastBackup(now(), bytes.size.toLong())
    }

    suspend fun remoteInfo(): DriveBackupInfo? = drive.find(DRIVE_BACKUP_FILE_NAME)?.toInfo()

    suspend fun restore(): RestoreSummary = lock.withLock {
        val file = drive.find(DRIVE_BACKUP_FILE_NAME) ?: throw DriveError.NoBackupFound()
        val text = drive.download(file.id)
        val summary = backup.importBackup(text).getOrElse { throw it }
        // Local and Drive now hold the same data, which is what LETS automatic backups start:
        // until this point an upload would have been this device's data overwriting someone else's.
        prefs.setLastBackup(now(), text.encodeToByteArray().size.toLong())
        summary
    }

    /**
     * Clears locally first, hands the grant back to Google afterwards — in that order because the
     * local half always succeeds and the remote half needs the network. Someone disconnecting on a
     * plane should see it disconnect.
     */
    suspend fun disconnect() {
        val token = runCatching { auth.accessToken() }.getOrNull()
        prefs.clearAccount()
        prefs.setFrequency(BackupFrequency.OFF)
        auth.clearSession()
        if (token != null) withTimeoutOrNull(5_000) { drive.revoke(token) }
    }
}
```

### 5.6 `AutoBackupRunner.kt` — the single "is it due?" decision

```kotlin
enum class AutoBackupResult { Done, Skipped, RetryLater, Failed }

/**
 * A scheduler that wakes slightly early still counts as due. WorkManager runs periodic work inside
 * a flex window rather than on the dot, and iOS decides for itself. Without slack, a run arriving
 * twenty minutes early would do nothing and wait a whole cadence for the next one.
 */
private const val DUE_SLACK_MS = 2 * 60 * 60 * 1000L

fun nextDueMillis(last: Long, f: BackupFrequency) = last + f.days * DAY_MS - DUE_SLACK_MS

class AutoBackupRunner(
    private val repo: DriveBackupRepository,
    private val prefs: DrivePrefs,
) {
    /** A background wake-up and an app launch can land together; only one of them uploads. */
    private val mutex = Mutex()

    suspend fun runIfDue(): AutoBackupResult = mutex.withLock {
        val s = prefs.snapshot()
        val last = s.lastBackupMillis
        when {
            s.account == null -> return AutoBackupResult.Skipped
            s.frequency == BackupFrequency.OFF -> return AutoBackupResult.Skipped

            // Never backed up or restored on THIS install. The account may hold another device's
            // backup the person has not decided about yet — and uploading a fresh install's empty
            // database over it is the one thing that cannot be undone. The first move is theirs.
            last == null -> return AutoBackupResult.Skipped

            now() < nextDueMillis(last, s.frequency) -> return AutoBackupResult.Skipped
        }
        try {
            repo.backupNow()
            AutoBackupResult.Done
        } catch (e: CancellationException) {
            throw e                                  // never swallow this
        } catch (e: DriveError) {
            when {
                e is DriveError.Network -> AutoBackupResult.RetryLater
                e is DriveError.Http && (e.code == 429 || e.code >= 500) -> AutoBackupResult.RetryLater
                // NeedsConsent, Auth, NotConnected: no amount of retrying produces a permission.
                else -> AutoBackupResult.Failed
            }
        } catch (e: Exception) {
            AutoBackupResult.Failed
        }
    }
}
```

Every path — WorkManager, iOS background task, app launch — comes through `runIfDue()`. The rules
are enforced once, not three times.

### 5.7 `DriveStartup.kt` — the safety net that actually carries the feature

```kotlin
/**
 * Two things, and the second is the important one. Re-applying the schedule puts back anything the
 * system dropped — a force-stop on Android clears pending work. Then a due backup is taken there
 * and then, which on iOS is the only DEPENDABLE backup there is: BGTaskScheduler runs when iOS
 * decides, and it may decide not to for days.
 */
class DriveStartup(
    private val prefs: DrivePrefs,
    private val scheduler: AutoBackupScheduler,
    private val runner: AutoBackupRunner,
    private val appScope: CoroutineScope,   // app-lifetime, NOT a screen's
) {
    fun onForeground() {
        appScope.launch {
            scheduler.apply(prefs.frequency())
            runner.runIfDue()
        }
    }
}
```

Call it from the app root:

```kotlin
LifecycleStartEffect(driveStartup) {
    driveStartup.onForeground()
    onStopOrDispose { }
}
```

`LifecycleStartEffect`, not `LaunchedEffect(Unit)` — the latter fires once per composition, so an
app resumed from the background after four days would never catch up.

### 5.8 `DriveModule.kt` — Koin wiring

```kotlin
object DriveQualifiers {
    val AppScope = named("drive_app_scope")   // outlives any screen
    val Engine   = named("drive_engine")      // OkHttp on Android, Darwin on iOS
    val Http     = named("drive_http")        // Drive's own client, never the app's
}

val driveModule: Module = module {
    single<CoroutineScope>(DriveQualifiers.AppScope) {
        CoroutineScope(SupervisorJob() + Dispatchers.Default)
    }
    single(DriveQualifiers.Http) { createDriveHttpClient(get(DriveQualifiers.Engine)) }
    single { DriveApiClient(http = get(DriveQualifiers.Http), auth = get()) }
    single { DrivePrefs(store = get()) }
    single { DriveBackupRepository(drive = get(), auth = get(), prefs = get(), backup = get()) }
    single { AutoBackupRunner(repo = get(), prefs = get()) }
    single { DriveStartup(prefs = get(), scheduler = get(), runner = get(),
                          appScope = get(DriveQualifiers.AppScope)) }
}
```

Also needed: `AutoBackupScheduler` (one method, `suspend fun apply(frequency)`) and
`fun parseRfc3339(value: String): Long? = runCatching { Instant.parse(value).toEpochMilliseconds() }.getOrNull()`.

---

## 6. The platform code

### 6.1 Android — authorization

Use `AuthorizationClient`. The older `GoogleSignIn` client is deprecated, and it is a *sign-in* API,
which an app with no accounts has no use for.

```kotlin
class AndroidGoogleAuthProvider(
    private val appContext: Context,
    private val activityProvider: ActivityProvider,
    private val prefs: DrivePrefs,
) : GoogleAuthProvider {

    private val scopes = listOf(Scope(DRIVE_APPDATA_SCOPE))

    override suspend fun connect(): String {
        val activity = activityProvider.current()
            ?: throw DriveError.Auth("No activity on screen to ask on")
        val client = Identity.getAuthorizationClient(activity)

        // No setAccount: leaving it out is what produces the ACCOUNT PICKER, which is also how
        // someone switches to a different account later.
        val request = AuthorizationRequest.builder().setRequestedScopes(scopes).build()

        val first = gms { client.authorize(request).await() }
        if (!first.hasResolution()) return first.tokenOrThrow()   // already granted, no screen

        val pending = first.pendingIntent ?: throw DriveError.NeedsConsent()
        val result = activity.launchForResult(pending)
        val data = result.data
        if (result.resultCode != Activity.RESULT_OK || data == null) throw DriveError.Cancelled()
        return gms { client.getAuthorizationResultFromIntent(data) }.tokenOrThrow()
    }

    override suspend fun accessToken(): String {
        val email = prefs.connectedEmail() ?: throw DriveError.NotConnected()
        val request = AuthorizationRequest.builder()
            .setRequestedScopes(scopes)
            // Pinned to the connected account: WITHOUT THIS, a device with several Google accounts
            // could silently return a token for a different one, and back up into its folder.
            .setAccount(Account(email, "com.google"))
            .build()
        val result = gms { Identity.getAuthorizationClient(appContext).authorize(request).await() }
        if (result.hasResolution()) throw DriveError.NeedsConsent()  // wants a screen; there is none
        return result.tokenOrThrow()
    }

    override suspend fun invalidate(token: String) {
        runCatching {
            Identity.getAuthorizationClient(appContext)
                .clearToken(ClearTokenRequest.builder().setToken(token).build()).await()
        }
    }

    override suspend fun clearSession() = Unit  // nothing local; revoke is shared REST

    private fun AuthorizationResult.tokenOrThrow() = accessToken ?: throw DriveError.NeedsConsent()

    private inline fun <T> gms(block: () -> T): T = try { block() } catch (e: ApiException) {
        throw when (e.statusCode) {
            CommonStatusCodes.NETWORK_ERROR -> DriveError.Network(e)
            CommonStatusCodes.CANCELED      -> DriveError.Cancelled()
            // DEVELOPER_ERROR (10) is worth recognising by sight: the build's signing certificate
            // or package name matches no OAuth client in the Cloud project.
            else -> DriveError.Auth("ApiException ${e.statusCode}")
        }
    }
}
```

Two helpers make this work from a platform-free ViewModel:

```kotlin
/**
 * Whichever Activity is on screen, for the one thing that needs one. Weakly held so a rotation
 * cannot leave a destroyed Activity alive in a singleton.
 */
class ActivityProvider {
    private var ref: WeakReference<ComponentActivity>? = null
    fun attach(a: ComponentActivity) { ref = WeakReference(a) }
    fun detach(a: ComponentActivity) { if (ref?.get() === a) ref = null }
    fun current(): ComponentActivity? = ref?.get()
}

/**
 * Launches the consent screen and suspends until it returns. The THREE-argument register is the one
 * without a LifecycleOwner — the ordinary registerForActivityResult must be called before the
 * Activity starts, and this is called when someone taps a button.
 */
suspend fun ComponentActivity.launchForResult(pending: PendingIntent): ActivityResult =
    withContext(Dispatchers.Main) {
        suspendCancellableCoroutine { cont ->
            lateinit var launcher: ActivityResultLauncher<IntentSenderRequest>
            launcher = activityResultRegistry.register(
                // Unique per call: a stale registration under a reused key would deliver this
                // result to the previous caller's callback.
                "drive_auth_${System.nanoTime()}",
                ActivityResultContracts.StartIntentSenderForResult(),
            ) { result -> launcher.unregister(); if (cont.isActive) cont.resume(result) }
            launcher.launch(IntentSenderRequest.Builder(pending.intentSender).build())
            cont.invokeOnCancellation { launcher.unregister() }
        }
    }
```

In your Activity: `activityProvider.attach(this)` in `onCreate`, `detach(this)` in `onDestroy`.

### 6.2 Android — scheduling

```kotlin
class AndroidAutoBackupScheduler(private val context: Context) : AutoBackupScheduler {
    override suspend fun apply(frequency: BackupFrequency) {
        val wm = WorkManager.getInstance(context)
        if (frequency == BackupFrequency.OFF) {
            wm.cancelUniqueWork(AutoBackupWorker.UNIQUE_NAME); return
        }
        // A FLEX WINDOW, not a bare period. Without one the flex equals the whole interval and
        // WorkManager may run a "daily" backup at any point in the day — 20 hours after the last
        // one, or 28. Asking for the final stretch keeps it near the same time of day.
        val request = PeriodicWorkRequestBuilder<AutoBackupWorker>(
            frequency.days.toLong(), TimeUnit.DAYS,
            if (frequency == BackupFrequency.WEEKLY) 12L else 2L, TimeUnit.HOURS,
        )
            .setConstraints(Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED).build())
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30L, TimeUnit.MINUTES)
            .build()

        // UPDATE, not REPLACE. This runs on every launch, and REPLACE would restart the period each
        // time — an app opened daily would NEVER reach a weekly backup.
        wm.enqueueUniquePeriodicWork(
            AutoBackupWorker.UNIQUE_NAME, ExistingPeriodicWorkPolicy.UPDATE, request,
        )
    }
}

/**
 * Decides nothing — AutoBackupRunner does. Dependencies come through KoinComponent rather than a
 * custom WorkerFactory: the default factory constructs workers reflectively and only ever passes
 * these two arguments.
 */
class AutoBackupWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params), KoinComponent {

    private val runner: AutoBackupRunner by inject()

    override suspend fun doWork(): Result = when (runner.runIfDue()) {
        // Skipped is a SUCCESS: not being due, or not being connected, is not a failure.
        AutoBackupResult.Done, AutoBackupResult.Skipped -> Result.success()
        AutoBackupResult.RetryLater ->
            if (runAttemptCount + 1 < 3) Result.retry() else Result.success()
        AutoBackupResult.Failed -> Result.failure()   // periodic work keeps its schedule anyway
    }

    companion object { const val UNIQUE_NAME = "drive_auto_backup" }
}
```

### 6.3 iOS — the Kotlin↔Swift bridge

Kotlin declares, Swift implements:

```kotlin
// iosMain
interface GoogleAuthBridge {
    fun requestDriveAccess(scopes: List<String>, completion: (String?, String?) -> Unit)
    fun freshToken(scopes: List<String>, completion: (String?, String?) -> Unit)
    fun clearSession()
}
```

`completion` gets exactly one non-null argument: a token, or an error code string. Those codes are
the contract between the two languages.

```kotlin
class IosGoogleAuthProvider(
    private val bridge: GoogleAuthBridge,
    private val prefs: DrivePrefs,
) : GoogleAuthProvider {

    private val scopes = listOf(DRIVE_APPDATA_SCOPE)

    // Everything on the bridge ends in UIKit, which is main-thread only.
    override suspend fun connect(): String =
        withContext(Dispatchers.Main) { bridgeCall { bridge.requestDriveAccess(scopes, it) } }

    override suspend fun accessToken(): String {
        if (prefs.connectedEmail() == null) throw DriveError.NotConnected()
        return withContext(Dispatchers.Main) { bridgeCall { bridge.freshToken(scopes, it) } }
    }

    /** The SDK refreshes its own token before it expires, so there is no cache here to clear. */
    override suspend fun invalidate(token: String) = Unit

    override suspend fun clearSession() { withContext(Dispatchers.Main) { bridge.clearSession() } }

    private suspend fun bridgeCall(call: ((String?, String?) -> Unit) -> Unit): String =
        suspendCancellableCoroutine { cont ->
            call { token, code ->
                if (cont.isActive) {
                    if (token != null) cont.resume(token)
                    else cont.resumeWithException(code.toDriveError())
                }
            }
        }

    private fun String?.toDriveError(): DriveError = when (this) {
        "CANCELLED"     -> DriveError.Cancelled()
        "NOT_CONNECTED" -> DriveError.NotConnected()
        "NEEDS_CONSENT" -> DriveError.NeedsConsent()
        "NETWORK"       -> DriveError.Network()
        "NO_CLIENT_ID"  -> DriveError.Auth("Google client ID is not configured")
        else -> {
            println("Drive: bridge returned an unhandled code — $this")
            DriveError.Auth(this ?: "UNKNOWN")
        }
    }
}
```

Swift side:

```swift
import UIKit
import GoogleSignIn
import ComposeApp

final class GoogleAuthBridgeImpl: NSObject, GoogleAuthBridge {

    func requestDriveAccess(scopes: [String], completion: @escaping (String?, String?) -> Void) {
        guard Self.hasClientID() else { completion(nil, "NO_CLIENT_ID"); return }
        guard let presenter = Self.topViewController() else { completion(nil, "NO_PRESENTER"); return }
        let gid = GIDSignIn.sharedInstance

        // Already has a session but not this scope — ask only for what is missing, so the person
        // is not made to pick their account again.
        if let user = gid.currentUser {
            let missing = scopes.filter { !(user.grantedScopes ?? []).contains($0) }
            guard !missing.isEmpty else { Self.fresh(user, completion); return }
            user.addScopes(missing, presenting: presenter) { result, error in
                Self.finish(result?.user, error, scopes, completion)
            }
            return
        }
        gid.signIn(withPresenting: presenter, hint: nil, additionalScopes: scopes) { result, error in
            Self.finish(result?.user, error, scopes, completion)
        }
    }

    func freshToken(scopes: [String], completion: @escaping (String?, String?) -> Void) {
        guard Self.hasClientID() else { completion(nil, "NO_CLIENT_ID"); return }
        let gid = GIDSignIn.sharedInstance
        if let user = gid.currentUser { Self.checked(user, scopes, completion); return }
        guard gid.hasPreviousSignIn() else { completion(nil, "NOT_CONNECTED"); return }
        // A relaunch, or a background task: the session is in the keychain, not in memory.
        gid.restorePreviousSignIn { user, error in
            guard let user else { completion(nil, error.map(Self.code(for:)) ?? "NOT_CONNECTED"); return }
            Self.checked(user, scopes, completion)
        }
    }

    func clearSession() { GIDSignIn.sharedInstance.signOut() }

    /// Signing in and being GRANTED the scope are two different things: the consent screen asks
    /// for it with a tick box, and Google hands back a valid session whether or not it stays
    /// ticked. So a sign-in result goes through the same check as a restored one.   <-- TRAP 2
    private static func finish(_ user: GIDGoogleUser?, _ error: Error?, _ scopes: [String],
                               _ completion: @escaping (String?, String?) -> Void) {
        if let error { completion(nil, code(for: error)); return }
        guard let user else { completion(nil, "UNKNOWN"); return }
        checked(user, scopes, completion)
    }

    private static func checked(_ user: GIDGoogleUser, _ scopes: [String],
                                _ completion: @escaping (String?, String?) -> Void) {
        let granted = user.grantedScopes ?? []
        guard scopes.allSatisfy({ granted.contains($0) }) else {
            print("Drive: scope not granted — wanted \(scopes) got \(granted)")
            completion(nil, "NEEDS_CONSENT"); return
        }
        fresh(user, completion)
    }

    private static func fresh(_ user: GIDGoogleUser,
                              _ completion: @escaping (String?, String?) -> Void) {
        user.refreshTokensIfNeeded { user, error in
            if let error { completion(nil, code(for: error)); return }
            guard let token = user?.accessToken.tokenString else { completion(nil, "UNKNOWN"); return }
            completion(token, nil)
        }
    }

    private static func code(for error: Error) -> String {
        let ns0 = error as NSError
        // Keep this. When something fails in a way you did not anticipate, it is the only evidence.
        print("Drive: GoogleSignIn failed — domain=\(ns0.domain) code=\(ns0.code) \(ns0.localizedDescription)")
        if let gidError = error as? GIDSignInError {
            switch gidError.code {
            case .canceled:            return "CANCELLED"
            case .hasNoAuthInKeychain: return "NOT_CONNECTED"
            default:                   return "GID_\(gidError.code.rawValue)"
            }
        }
        let ns = error as NSError
        if ns.domain == NSURLErrorDomain { return "NETWORK" }
        return "UNKNOWN_\(ns.domain)_\(ns.code)"
    }

    /// A missing or placeholder client id makes GoogleSignIn raise an OBJECTIVE-C EXCEPTION, which
    /// Swift cannot catch — so it takes the app down rather than failing the call.   <-- TRAP 3
    private static func hasClientID() -> Bool {
        guard let id = Bundle.main.object(forInfoDictionaryKey: "GIDClientID") as? String
        else { return false }
        return !id.isEmpty && !id.hasPrefix("REPLACE_WITH_")
    }

    /// Whatever is actually in front of the person — the Compose host, or a sheet above it.
    private static func topViewController() -> UIViewController? {
        let scene = UIApplication.shared.connectedScenes
            .compactMap { $0 as? UIWindowScene }
            .first { $0.activationState == .foregroundActive }
        var top = scene?.keyWindow?.rootViewController
        while let presented = top?.presentedViewController { top = presented }
        return top
    }
}
```

Handing it to Kotlin — the bridge must exist **before Koin is built**:

```kotlin
// iosMain/di/PlatformModule.ios.kt
private var authBridge: GoogleAuthBridge? = null

/** Swift: PlatformModule_iosKt.registerGoogleAuthBridge(bridge:), from iOSApp.init() */
fun registerGoogleAuthBridge(bridge: GoogleAuthBridge) { authBridge = bridge }
```

```swift
@main
struct iOSApp: App {
    init() {
        // Both before the first view exists, and in this order. The bridge has to be in place
        // before Koin is built, and iOS refuses a background task identifier registered after
        // launch finishes.
        PlatformModule_iosKt.registerGoogleAuthBridge(bridge: GoogleAuthBridgeImpl())
        IosAutoBackupKt.registerAutoBackupTask()
    }

    var body: some Scene {
        WindowGroup {
            ContentView().onOpenURL { url in
                if url.isFileURL { openBackupFile(url) }
                else { GIDSignIn.sharedInstance.handle(url) }   // the consent page coming back
            }
        }
    }
}
```

### 6.4 iOS — scheduling

```kotlin
/** Must match BGTaskSchedulerPermittedIdentifiers in Info.plist EXACTLY, or iOS refuses it. */
const val AUTO_BACKUP_TASK_ID = "com.yourcompany.yourapp.autobackup"

/** iOS rejects a request asking to run immediately; this is the floor it will accept. */
private const val MIN_DELAY_MS = 15 * 60 * 1000L

class IosAutoBackupScheduler(private val prefs: DrivePrefs) : AutoBackupScheduler {
    override suspend fun apply(frequency: BackupFrequency) {
        val scheduler = BGTaskScheduler.sharedScheduler
        if (frequency == BackupFrequency.OFF) {
            scheduler.cancelTaskRequestWithIdentifier(AUTO_BACKUP_TASK_ID); return
        }
        val now = now()
        val due = prefs.lastBackupMillis()?.let { nextDueMillis(it, frequency) } ?: now
        val request = BGProcessingTaskRequest(AUTO_BACKUP_TASK_ID).apply {
            requiresNetworkConnectivity = true
            requiresExternalPower = false     // one small JSON file; no need to wait for a charger
            earliestBeginDate = NSDate.dateWithTimeIntervalSinceNow(
                maxOf(due - now, MIN_DELAY_MS) / 1000.0)
        }
        // Submitting the same identifier replaces whatever was pending, so this is safe on launch.
        memScoped {
            val error = alloc<ObjCObjectVar<NSError?>>()
            if (!scheduler.submitTaskRequest(request, error.ptr)) {
                // ALWAYS fails on the simulator, which has no background scheduling at all.
                println("BGTask submit failed — ${error.value?.localizedDescription}")
            }
        }
    }
}

fun registerAutoBackupTask() {
    BGTaskScheduler.sharedScheduler.registerForTaskWithIdentifier(AUTO_BACKUP_TASK_ID, null) { task ->
        (task as? BGProcessingTask)?.let(::handleAutoBackup)
    }
}

private fun handleAutoBackup(task: BGProcessingTask) {
    var ok = false
    val job = deps.appScope.launch(start = CoroutineStart.LAZY) {
        // Apple's advice: book the NEXT slot first. If this work runs long and is killed, the
        // following one is already scheduled.
        deps.scheduler.apply(deps.prefs.frequency())
        val result = deps.runner.runIfDue()
        ok = result == AutoBackupResult.Done || result == AutoBackupResult.Skipped
    }
    task.expirationHandler = { job.cancel() }
    // EXACTLY ONCE, however the job ended. Calling setTaskCompleted twice, or not at all, is how
    // an app teaches iOS to stop granting it background time.
    job.invokeOnCompletion { cause -> task.setTaskCompletedWithSuccess(cause == null && ok) }
    job.start()
}
```

`IosAutoBackupScheduler` needs `@OptIn(ExperimentalForeignApi::class, BetaInteropApi::class)`.

### 6.5 Platform DI

```kotlin
// androidMain
single { ActivityProvider() }
single<HttpClientEngine>(DriveQualifiers.Engine) { OkHttp.create() }
single<GoogleAuthProvider> {
    AndroidGoogleAuthProvider(androidContext(), activityProvider = get(), prefs = get())
}
single<AutoBackupScheduler> { AndroidAutoBackupScheduler(androidContext()) }

// iosMain
single<HttpClientEngine>(DriveQualifiers.Engine) { Darwin.create() }
single<GoogleAuthProvider> {
    val bridge = authBridge ?: error("call registerGoogleAuthBridge() in iOSApp.init()")
    IosGoogleAuthProvider(bridge = bridge, prefs = get())
}
single<AutoBackupScheduler> { IosAutoBackupScheduler(prefs = get()) }
```

---

## 7. Every trap, and how to avoid it

These all happened. In rough order of how much time each one costs.

### Trap 1 — iOS simulator builds are unsigned, so the keychain is closed

**Symptom.** Sign-in completes, the person picks their account, and then the app is still not
connected. No consent checkbox ever appears. The error is generic.

**The evidence, once you log it:** `domain=com.google.GIDSignIn code=-2 keychain error`.

**Cause.** A common KMP `project.yml` carries:

```yaml
"CODE_SIGNING_ALLOWED[sdk=iphonesimulator*]": "NO"
```

because simulator builds "need no signing". But an unsigned app carries **no entitlements**, and
since iOS 15 an app with no entitlements cannot touch the keychain. GoogleSignIn stores its session
there. Device builds are signed normally, so this is invisible until someone tests on a simulator —
which is where you test.

**Fix.** Sign the simulator ad-hoc, which still needs no team and no certificate:

```yaml
CODE_SIGN_ENTITLEMENTS: iosApp/iosApp.entitlements
"CODE_SIGN_IDENTITY[sdk=iphonesimulator*]": "-"
"CODE_SIGNING_ALLOWED[sdk=iphonesimulator*]": "YES"
"CODE_SIGNING_REQUIRED[sdk=iphonesimulator*]": "YES"
```

`iosApp/iosApp.entitlements`:

```xml
<key>keychain-access-groups</key>
<array><string>$(AppIdentifierPrefix)$(PRODUCT_BUNDLE_IDENTIFIER)</string></array>
```

`$(PRODUCT_BUNDLE_IDENTIFIER)`, **not** `$(CFBundleIdentifier)` — the latter is not a build setting
and silently resolves to nothing, leaving the entitlements dictionary empty.

**Verify it, don't assume it:**

```bash
plutil -p <DerivedData>/Build/Intermediates.noindex/iosApp.build/Debug-iphonesimulator/\
iosApp.build/iosApp.app-Simulated.xcent
```

You must see both keys:

```
"application-identifier" => "TEAMID.com.yourcompany.yourapp"
"keychain-access-groups" => [ 0 => "TEAMID.com.yourcompany.yourapp" ]
```

An empty `{}` means it did not take.

### Trap 2 — signing in is not the same as being granted the scope

Google's consent screen shows the Drive permission as a **tick box**. Untick it and sign-in still
succeeds, and `GIDSignIn` hands back a perfectly valid session and a real access token. The token
simply cannot touch Drive.

If you take the token straight from the sign-in callback, the app believes it is connected and then
fails at the first Drive call — reported as whatever that call's HTTP error happens to be, which
tells you nothing.

**Fix.** Check `user.grantedScopes` after sign-in, not only after restoring a session. See
`finish()` above. Also keep the 403-with-"insufficient" check in `authed()` as a backstop for the
case where the scope is revoked from the web later.

### Trap 3 — a missing `GIDClientID` crashes the app, uncatchably

GoogleSignIn raises an **Objective-C exception** when `GIDClientID` is absent or a placeholder.
Swift cannot catch that. The app dies the moment someone taps Connect.

**Fix.** The `hasClientID()` guard above. An app that says "not set up yet" beats an app that
disappears.

### Trap 4 — the 401 retry masks the real error on the connect path

`authed()` retries a 401 by asking the auth provider for a fresh token. On the **connect** path
that is wrong: the token was handed in by `connect()` and is the freshest one that will ever exist,
and the account has not been saved to prefs yet — so `accessToken()` throws `NotConnected`.

The person then reads *"Connect a Google account first"* while standing on the Connect button.

**Fix.** `if (initialToken != null) throw DriveError.NeedsConsent()` before the retry.

### Trap 5 — `ExistingPeriodicWorkPolicy.REPLACE` means weekly backups never happen

You re-apply the schedule on every launch (correctly — a force-stop clears pending work). With
`REPLACE`, every launch restarts the period. An app opened daily never reaches a weekly backup.

**Fix.** `UPDATE`. It picks up a changed cadence without resetting the running timer.

### Trap 6 — periodic work with no flex window drifts across the whole day

`PeriodicWorkRequestBuilder(1, DAYS)` sets the flex interval equal to the whole period. WorkManager
may run a "daily" backup 20 hours after the last one, or 28. It feels random, because it is.

**Fix.** Pass an explicit flex: `(1, DAYS, 2, HOURS)`. And give `AutoBackupRunner` a matching slack
(2h) so a run arriving early inside that window still counts as due — otherwise it does nothing and
waits another full period.

### Trap 7 — `BGTaskScheduler` will teach you to distrust it

- It **always** fails to submit on the simulator. Log it, do not throw.
- The identifier must appear in `BGTaskSchedulerPermittedIdentifiers` **character for character**.
- `registerForTaskWithIdentifier` must be called before launch finishes — in `init()`, not in a view.
- `setTaskCompleted` must be called **exactly once**. Twice, or never, and iOS quietly stops giving
  your app background time.
- Even done perfectly, iOS may not run it for days.

**Therefore:** never let iOS auto-backup depend on `BGTaskScheduler`. The real mechanism is
`DriveStartup.onForeground()` — catch up on launch. Treat the background task as a bonus.

### Trap 8 — Google's Drive client library is JVM-only

`com.google.api-client` / `google-api-services-drive` will not link on Kotlin/Native. Discovering
this after building Android first means rewriting the whole data layer.

**Fix.** Plain REST from the start, as in §5.3. Six endpoints.

### Trap 9 — silent token for the wrong Google account

On Android, `AuthorizationRequest` **without** `setAccount()` returns a token for whichever account
Play Services feels like. On a phone with a personal and a work account, backups can start landing
in the wrong folder, and nothing announces it.

**Fix.** Omit `setAccount` in `connect()` (you *want* the picker), and always pin it in
`accessToken()` using the email you saved.

### Trap 10 — `ApiException` status code 10

`DEVELOPER_ERROR`. It means the package name or signing certificate of this build matches no OAuth
client in the Cloud project. Nine times out of ten: you created the client with the release SHA-1
and are running a debug build, or vice versa. Learn to recognise the number.

### Trap 11 — publishing status, and the 7-day disappearing connection

In *Testing*, refresh tokens expire after 7 days. Testers report the app "forgets" Drive every week.
Also, iOS enforces the test-user allowlist much more strictly than Android's Play Services path —
so the classic shape of this bug is "Android works, iOS doesn't".

**Fix.** Publish to production. With only `drive.appdata` it needs no review.

### Trap 12 — XcodeGen's `info:` block overwrites your committed `Info.plist`

If `project.yml` uses an `info:` block, XcodeGen **generates** the plist at that path, destroying
whatever was there — `GIDClientID`, URL schemes, `CADisableMinimumFrameDurationOnPhone` (without
which Compose 1.12 aborts on launch), your display name — every time anyone runs `xcodegen generate`.

**Fix.** `INFOPLIST_FILE: iosApp/Info.plist` and keep the plist under version control.

### Trap 13 — the OAuth redirect never comes back

`CFBundleURLSchemes` must contain the **reversed** client ID
(`com.googleusercontent.apps.NNN-hash`), and `onOpenURL` must route non-file URLs to
`GIDSignIn.sharedInstance.handle(url)`. Miss either and sign-in hangs with no error at all — which
is harder to diagnose than a crash.

### Trap 14 — auto-backup overwriting another device's backup

New phone. Person installs the app, connects the account their old phone was backing up to, and
hasn't restored yet. An automatic backup fires and uploads the empty database over the only copy of
their data.

**Fix.** The `last == null -> Skipped` rule in `AutoBackupRunner`. Automatic backups only begin once
*this install* has either backed up or restored at least once. `connect()` backs up immediately only
when the folder is **empty**, so most people never notice the rule exists.

### Trap 15 — a backup running during a restore

An auto-backup firing mid-restore uploads the half-replaced database, over the very file being
restored from.

**Fix.** One `Mutex` in `DriveBackupRepository` covering `backupNow` and `restore`, plus a second in
`AutoBackupRunner` so a background wake-up and a launch cannot both start.

### Trap 16 — disconnecting fails offline

Revoke first and the whole disconnect fails without a network. Someone disconnecting on a plane
watches the account stay connected.

**Fix.** Clear local state first, revoke afterwards, wrapped in `withTimeoutOrNull(5_000)`.

### Trap 17 — `expectSuccess = true`

Ktor's default throws on non-2xx, so `authed()` never sees the 401 it exists to handle. Set
`expectSuccess = false` on the Drive client.

### Trap 18 — Drive sends int64 as a string

`size` is `"6312"`, not `6312`. Type it `String?` and use `toLongOrNull()`. Also set
`ignoreUnknownKeys = true` — Drive adds fields.

### Trap 19 — Compose Resources does not unescape `\'`

`<string name="x">Couldn\'t read the file</string>` puts a literal backslash on screen. Use a
typographic apostrophe (`’`) instead.

### Trap 20 — a version catalog alias ending in a Kotlin keyword

`compose-material3-window-size-class` generates the accessor
`libs.compose.material3.window.size.class` — and `class` is a keyword. The build fails with a
message that does not mention your alias. Rename it.

### Trap 21 — `LaunchedEffect(Unit)` instead of `LifecycleStartEffect`

`LaunchedEffect(Unit)` fires once per composition. An app resumed from the background four days
later never re-checks whether a backup is due. Use `LifecycleStartEffect`.

### Trap 22 — one iOS client is enough; two is a trap

Because Android needs a debug client and a release client, people assume iOS needs the same. It does
not — iOS clients are identified by bundle ID alone. A second "prod" iOS client with the same bundle
ID is byte-for-byte equivalent and will only make you wonder which ID belongs in `Info.plist`.

---

## 8. Test checklist

Work down this list. Items marked **[both]** must be done on Android *and* iOS.

**Connect**
- [both] Connect on a clean install → account email appears, first backup taken
- [both] Connect to an account that **already has** a backup → app *reports* it, does not overwrite
- [both] Cancel the account picker → no error toast, nothing changes
- [both] Untick the Drive box on the consent screen → "Drive permission needed", not a generic error
- [both] Connect with no network → "Check your internet connection"

**Backup / restore**
- [both] Back up now → size and timestamp update
- [both] Back up twice → the account still holds exactly **one** file
- [both] Restore → data returns; restore with an empty account → "No backup in this account"
- [both] Force-quit mid-upload → next launch recovers, no corrupt file

**Cadence**
- [both] Set Daily, force-quit, relaunch → schedule re-applied (Android: `adb shell dumpsys jobscheduler | grep <pkg>`)
- Android: `adb shell cmd jobscheduler run -f <pkg> <id>` to force a run
- iOS: pause in the debugger and use the `_simulateLaunchForTaskWithIdentifier` LLDB trick
- [both] Set Off → pending work cancelled
- [both] Turn the clock forward past the due time, relaunch → catch-up backup fires

**Disconnect**
- [both] Disconnect → confirmation dialog first, then account cleared, cadence Off
- [both] Disconnect with no network → still disconnects locally
- [both] Check [myaccount.google.com/permissions](https://myaccount.google.com/permissions) → grant gone

**Accounts and builds**
- Android: two Google accounts on the device → backups always land in the connected one
- Android: release-signed build with the Play SHA-1 client → connects (this is where `ApiException 10` shows up)
- iOS: simulator **and** a real device
- [both] Revoke access from the web, then open the app → "Drive permission needed", recoverable by reconnecting

---

## Appendix A — encrypting the backup file

Optional, and worth doing: a backup is a complete record of someone's data, and it ends up in
Downloads, in a chat, on a memory card.

**Be honest about the limit.** With no user password there is nowhere for a key to live except the
binary. This stops other apps and casual reading; it does not stop someone who decompiles your app.
Write that in the code comment so nobody later mistakes it for real protection.

Envelope:

```
MAGIC1.<base64 of: iv(16) || ciphertext || mac(32)>
```

AES-256-CBC for the content, HMAC-SHA-256 over `iv || ciphertext`, computed after encryption and
**verified before decryption** so a tampered or truncated file is rejected rather than decrypted
into nonsense. Compare the MAC in constant time.

Keep the shared logic in `commonMain` and `expect` only five primitives — `aesCbcEncrypt`,
`aesCbcDecrypt`, `hmacSha256`, `sha256`, `secureRandomBytes` — so the two platforms cannot drift
into producing files the other cannot read. Android: `javax.crypto`. iOS: CommonCrypto (`CCCrypt`,
`CCHmac`, `CC_SHA256`, `SecRandomCopyBytes`).

Two Kotlin/Native details that cost time:

- `CC_SHA256` needs `o.addressOf(0).reinterpret<UByteVar>()` — the pointer types do not match.
- An empty `ByteArray` has no address to pin; guard before `usePinned`.

On import, accept both sealed and plain files (`looksEncrypted(text)`), so backups made before you
shipped encryption still restore.

---

## Appendix B — file association

To make your exported file open only in your app:

**Android** — `AndroidManifest.xml`, a VIEW intent-filter for your vendor MIME type, plus
`pathPattern` variants for the extension (`.*\\.yourext`, `.*\\..*\\.yourext`, … — Android's
pathPattern does not backtrack, so you need several).

**iOS** — `UTExportedTypeDeclarations` (declare the type) *and* `CFBundleDocumentTypes` (claim it).
Both. In `onOpenURL`, file URLs arrive **security-scoped**: call
`startAccessingSecurityScopedResource()`, read, and release in a `defer` — skip it and the read
fails and the scope leaks.

On both platforms the incoming file should be handed to a `MutableStateFlow<String?>` that the UI
observes and turns into a confirmation dialog. Never restore just because a file was opened.

---

*Written from a shipped implementation. If something here contradicts what you observe, trust what
you observe — then fix this document.*
