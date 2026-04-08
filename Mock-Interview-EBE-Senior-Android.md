# Mock Interview: Senior Android Developer — EBE

**Role:** Senior Android Developer  
**Company:** EBE  
**Prepared:** April 8, 2026  

This document collects questions and drafted answers for interview preparation. Questions will be added gradually; answers are refined as we go.

**Note:** The answers below use **simple, clear English** (about **B1 level**). The goal is easier reading and speaking practice. Technical words (for example ViewModel, REST) stay when they are needed for the job.

---

## How to use

- Paste each new question in order (or note the section number).
- Answers below are starting points: add your real projects, numbers, and trade-offs before the real interview.

---

## Answering with ADD (Matt Abrahams, *Think Faster, Talk Smarter*)

Use **ADD** to open your answer, then use the **Short version** and bullets for detail.

| Step | What to do |
|------|------------|
| **A — Answer** | One **clear** sentence that answers the question. |
| **D — Detail** | A **concrete** example—**replace** **`_(your example)_`** with your real banking/Android work. |
| **D — Describe relevance** | Why it matters for **EBE** / **banking** / **Senior Android** expectations. |

Each question below includes an **ADD** block after **`**Answer (simple English, enhanced)**`**.

---

## Questions & answers

### Question 1

**Imagine you're working on a new banking feature in an app using MVVM and Clean Architecture—how would you structure the layers and handle state with Coroutines or Flow?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I use **Presentation → Domain → Data**: **ViewModel** calls **use cases**, **StateFlow** for screen state, **coroutines** + **Flow** with correct **dispatchers**—**domain** stays free of **Retrofit** / **Android** imports.  
- **D — Detail:** _(your example)_ **Repository** implements API + **Room**, **fake** use cases in **ViewModel** tests—*(banking screen you shipped)*.  
- **D — Describe relevance:** **EBE**-style **banking** needs **correct** **layering** and **safe** UI state—this shows **Clean** **Architecture** under real constraints.

**Short version (about 30 seconds)**  
I use **three main layers**: **Presentation** (UI + **ViewModel**), **Domain** (**use cases** + pure **models**), and **Data** (**repository** implementations + remote/local sources). The **ViewModel** talks only to **use cases**, not to Retrofit or Room directly. **State** is **immutable** and exposed with **`StateFlow`** (or **`SharedFlow`** for one-off events like errors/snacks). **Coroutines** run in **`viewModelScope`** with the right **`Dispatcher`**; **Flow** from the domain/data layers can be **`stateIn`** / **`shareIn`** in the ViewModel so the UI gets a simple, testable stream.

---

**1. Layers — who talks to whom**

| Layer | What lives here | Depends on |
|--------|-----------------|------------|
| **Presentation** | Fragments/Compose, **ViewModel** | Domain only (interfaces / use cases) |
| **Domain** | **Entities**, **use case** classes, repository **interfaces** | Nothing from Android framework or Retrofit |
| **Data** | Repository **implementations**, API (**Retrofit**), DB (**Room**), mappers **DTO → domain** | Framework OK here |

**MVVM in this picture:** **M** = domain models + use case outputs; **V** = UI; **VM** = coordinates user actions, calls use cases, holds **UI state**. The **ViewModel** does **not** own business rules that belong in the domain—those sit in **use cases** so we can unit-test without Android.

**Banking angle (one sentence):** Sensitive details stay in **data** and **server contracts**; the ViewModel exposes **safe UI state** (for example “authenticated,” “amount confirmed”), not raw tokens in logs.

---

**2. State — Flow vs Coroutines in the ViewModel**

- **`StateFlow`:** Best for **screen state** the UI **always** needs (loading, content, error). One **source of truth**; updates with **`value = copy(...)`** from a **data class** or **sealed** hierarchy.
- **`SharedFlow` / `Channel`:** For **one-time** events (navigation, one-off snackbars) so we do **not** replay old events after rotation—optional pattern.
- **Where Coroutines fit:** Use **`viewModelScope.launch`** to **collect** flows from use cases, or to **call suspend** functions. **Never** block the main thread.
- **`flowOn(Dispatchers.IO)`** (or IO inside the repository): network/DB work off the main thread. **Inject** dispatchers in tests (**`TestDispatcher`**).

**Words you can explain if asked:** **Use case** = one job the app does (for example `GetAccountBalance`); **repository interface** in domain, **implementation** in data.

---

**3. Clean Architecture “rules” I keep under pressure**

- **Dependency rule:** Inner layers do **not** know outer layers. Domain has **no** Retrofit imports.
- **Mappers at boundaries:** API **DTOs** stop in the data layer; domain sees **stable** models.
- **Single responsibility:** If a banking rule gets heavy (fees, eligibility), it gets its **own** use case instead of growing the ViewModel.

---

**4. Testing (short)**

- **Use cases:** pure Kotlin tests, fake repositories.
- **ViewModel:** fake use cases + **`TestDispatcher`**, assert **StateFlow** emissions.
- **Data:** **MockWebServer** / fakes for API contracts.

---

**5. Trade-off**  
Full Clean Architecture with **many** thin use cases can be **verbose** for tiny screens. For a **very** small feature, I might keep **one** use case or a **slim** repository interface—but I still **do not** skip **domain models** + **testable** boundaries in a **banking** app where correctness matters.

**One-line close:** *“Presentation depends on domain use cases; data implements repository contracts; ViewModel exposes immutable StateFlow built from Flow and coroutines—clear layers, safe banking UI state, testable core.”*

---

### Question 2 (follow-up)

**How would you ensure that the Coroutines in your ViewModel handle backpressure or high-frequency updates from data sources, especially under those tight deadlines you mentioned?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I **shape** hot streams with **`conflate`**, **`debounce`**, **`distinctUntilChanged`**, and **tight** **`shareIn`** buffers so the UI does not **flood** under **load**.  
- **D — Detail:** _(your example)_ **Balance** ticks or **price** updates **conflated** to **latest**; heavy mapping on **`Dispatchers.Default`**—*(your stream)*.  
- **D — Describe relevance:** **Banking** UIs must stay **smooth** at peak **without** **wrong** **balances**—**timing** can change, **truth** cannot.

**Short version (about 30 seconds)**  
**Kotlin Flow** already **slows the producer** when the collector is slow—**suspend** is the default backpressure story. When updates are **too fast for the UI**, I **shape** the stream: **`conflate()`** keeps only the **latest** value, **`debounce`** waits for a **pause**, **`distinctUntilChanged`** skips **duplicates**. For **shared** hot streams I configure **`shareIn`** / **`stateIn`** with a **small buffer** and **`onBufferOverflow = DROP_OLDEST`** (or **`LATEST`**) so we **do not** grow memory. Under a **deadline**, I fix the **noisiest** stream first—usually **one operator**—and push **heavy** mapping **down** to **`flowOn(Dispatchers.Default)`** so the main collector stays light.

---

**1. What “backpressure” means here**  
If a **sensor**, **WebSocket**, or **polling** job emits **very often**, the **ViewModel** must not **spam** the UI or **queue** endless work. We need a **clear rule**: either **process every** update (rare for UI), or **sample / merge** updates into something the user can **follow**.

---

**2. Flow’s built-in behavior**  
A **cold** `Flow` runs **only when collected**; each **`emit` suspends** until the collector is ready—so you get **natural** coordination. **Hot** streams (**SharedFlow**, **callbackFlow**, **channelFlow**) need **explicit** limits because they can **produce** faster than the UI should **consume**.

---

**3. Operators I reach for first (fast to ship)**

| Situation | Operator / pattern | Why |
|-----------|-------------------|-----|
| **Only the latest value matters** (balance tick, progress) | **`conflate()`** | Drops **middle** values; collector always gets the **newest**. |
| **User is typing / rapid taps** | **`debounce`** (and maybe **`distinctUntilChanged`**) | Fewer updates; avoids **storm** of API calls. |
| **Same value repeated** | **`distinctUntilChanged()`** | Cheap **dedupe**. |
| **“At most every N ms”** | **`sample`** (or **`chunked`** + take last) | Smooth **throttle** for charts/lists. |

**Deadline rule:** add **one** operator, **measure** (even roughly), then add more only if needed.

---

**4. `shareIn` / `stateIn` and buffers**  
When I **share** one upstream `Flow` to many collectors, I use **`shareIn`** / **`stateIn`** with:

- **`extraBufferCapacity`** small (often **0** or **1**),
- **`onBufferOverflow`:** **`DROP_OLDEST`** or **`SUSPEND`** depending on whether **losing** an old tick is OK (many **live** UIs: **latest wins**).

That stops a **hidden** queue from growing forever.

---

**5. Coroutines in the ViewModel**  
- **`launch`** per **collection** is fine; use **`supervisorJob`** patterns only if **multiple** independent streams must **not** cancel each other—otherwise keep **structured** scope simple.  
- **Heavy** work (big **JSON** parse, **large** list mapping): **`flowOn(Dispatchers.Default)`** or do it **in the repository**, not on **Main**.  
- **Never** **`collect`** an infinite hot flow on Main **without** **conflate** / **debounce** if it can **flood** composables.

---

**6. Banking / safety (short)**  
High frequency must **not** mean **inconsistent** balances: I still **map** to **domain** rules in **one** place; **conflate** affects **timing**, not **correctness** of the **last** committed state the user sees.

---

**7. Trade-off**  
**`conflate`** can **skip** intermediate states—good for **UI**, bad if you **audit** every tick. Then I **branch**: **full** stream to **logging/metrics** (careful with **PII**), **shaped** stream to the **screen**.

**One-line close:** *“Rely on Flow’s suspend for natural pacing; for hot or noisy sources, conflate/debounce/distinctUntilChanged and tight shareIn buffers so the ViewModel stays responsive without piling up work.”*

---

### Question 3

**Suppose you're dealing with a critical screen showing slow load times and occasional ANRs in production—how would you approach isolating the bottleneck using profiling tools, and what would be your first step in deciding what to optimize?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I **measure** with **Play** **ANRs**, **Firebase**, **CPU** **System** **Trace**, then **classify** the cost (**main** **CPU** vs **block** vs **network** vs **GC**) before I **optimize**.  
- **D — Detail:** _(your example)_ **Profiler** shows **parse** on **Main** → move to **Default**; **Macrobenchmark** on **cold** start—*(slow screen you fixed)*.  
- **D — Describe relevance:** **Senior** engineers **prove** **bottlenecks** with **data**—critical for **customer-facing** **banking** flows.

**Short version (about 30 seconds)**  
I **do not** guess—I **reproduce** and **measure**. I pull **ANR traces** from **Play Console** or **Firebase Crashlytics** (if linked) and look for **blocked main thread** (long frames, **Binder** waits, **locks**). Locally I use **Android Studio Profiler**: **CPU** (record a trace—**System Trace** / **Perfetto**-style) while opening the screen, plus **Memory** if I suspect **GC** pressure. **`Baseline Profiles`** / **`Macrobenchmark`** help **cold start** and **jank**. My **first optimization decision** comes from **where time goes**: if the **main thread** is busy → fix **layout**, **work on Main**, or **too much work per frame**; if the **main thread waits** → **I/O**, **locks**, or **slow** remote—then I move work or **parallelize** off Main. **Biggest user-visible win first** after I know the **bottleneck**.

---

**1. Separate “slow load” from “ANR”**  
- **Slow load:** time-to-first-content, **network** + **parsing** + **first draw**.  
- **ANR:** **main thread** did not respond ~**5s** (input) or **broadcast** path—often **heavy** work, **deadlock**, or **blocking** call on Main.

They can share a root cause (for example **parse on Main**), but the **tools** I emphasize first differ slightly.

---

**2. Production signals (first data I grab)**  
- **Google Play** → **ANRs & crashes** → download **stack traces** (often shows **main** blocked in `MessageQueue` or inside **your** code).  
- **Firebase Crashlytics** / internal **logging**: **custom keys** around the screen, **trace** IDs.  
- **Firebase Performance** (or **custom** **timing** events): **screen load** duration, **API** latency—**percentiles**, not only **average**.

This tells me **whether** the pain is **startup**, **network**, **after-data UI**, or **touch** stalls.

---

**3. Local profiling — what I run**

| Tool | What it shows |
|------|----------------|
| **Profiler → CPU** | **System Trace** / method sampling: **frame timeline**, **main-thread** work, **layout**, **inflate**, **Binder** time. |
| **Profiler → Memory** | **Allocations**, **GC** spikes during load; **leaks** (LeakCanary in **debug**). |
| **Layout Inspector** | Deep **view** hierarchy, **re-layout** suspects. |
| **Macrobenchmark** / **Baseline Profile** | **Startup** / **scroll** **jank** with **repeatable** numbers. |

For **Compose**, I also watch **recomposition** counts (Studio **Layout Inspector** / **Composition tracing**) if the trace points to **UI** work.

---

**4. How I isolate the bottleneck (order of thinking)**  
1. **Confirm** the slow path: **open screen** with **representative** data (large lists, bad network **profile**).  
2. **CPU trace**: Is **Choreographer#doFrame** or **layout** taking **big slices**? Is **my** package on Main for **milliseconds**?  
3. **Network**: **Retrofit** / **OkHttp** **event listener** or **server** timings—**wait** vs **download** vs **parse**.  
4. **Off-main check:** **StrictMode** in **debug** (disk/network on Main); **coroutine** dispatcher mistakes.

---

**5. First step in deciding *what* to optimize (the answer they asked for)**  
**First step:** classify the cost: **(A)** **main-thread CPU** (measure/layout/draw/compose), **(B)** **main-thread block** (**sync** I/O, **lock**, **sleep**), **(C)** **waiting** on **network** (often **not** an ANR by itself, but **feels** slow), **(D)** **memory/GC** stutter.

**Rule:** **User-perceived** **impact × confidence** from the trace. Example: trace shows **JSON parse** on Main → **move** to **`Dispatchers.Default`**—high confidence, **direct** ANR fix. If trace shows **flat** Main but **slow** API → **optimize** **backend** or **caching** / **pagination** first for **load time**, not **micro-optimizations** in **RecyclerView** **diffing**.

---

**6. Under pressure (deadlines)**  
I add **light** **telemetry** (screen **open** → **first** **meaningful** **paint**, **API** **done**) so the **next** release **proves** the fix. I avoid **random** **caching** without **before/after** numbers.

---

**7. Trade-off**  
**Heavy** **tracing** in **release** has **overhead**—I use **sampling** or **debuggable** builds for deep traces, and **minimal** **markers** in prod.

**One-line close:** *“Start from production ANR stacks and percentile timings, reproduce with CPU/System Trace to see main-thread vs I/O vs layout, then fix the category that the trace proves—biggest verified bottleneck first.”*

---

### Question 4 (follow-up)

**What strategy would you use to verify that your fix actually resolves the ANR without introducing regressions in other areas of the app?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I **compare** **before/after** **ANR** **rates** and **traces**, run **benchmarks** on **touched** **flows**, use **staged** **rollout** and **flags**, and **test** **nearby** **modules** that **share** **code**.  
- **D — Detail:** _(your example)_ **Play** **Console** **ANR** **by** **version**; **Macrobenchmark** on **login** **path**; **smoke** **suite**—*(your verification story)*.  
- **D — Describe relevance:** **EBE** needs **proof** the **ANR** **fix** **did not** **break** **other** **banking** **journeys**.

**Short version (about 30 seconds)**  
I treat it like a **small release**: **baseline** metrics **before** the fix, then **compare** **after** on the **same** **journeys**. I use **Play Console** / **Firebase** **ANR** and **crash** rates by **version**, plus **custom** **timings** for the **screen** we touched. **Locally** I **re-run** **CPU**/**Macrobenchmark** on the **hot path** and run **automated** **UI**/**integration** tests for **nearby** flows. I ship through **internal** → **beta** → **staged** rollout with **alerts** if **ANR** or **errors** **jump**. **Feature flags** help: I can **toggle** the fix for a **slice** of users and **watch** dashboards. For **regressions**, I focus on **shared** code we changed—**threading** and **ordering**—and add **tests** where the bug was **invisible** to the old **checks**.

---

**1. Prove the ANR is really gone (not “feels better”)**  
- **Production:** compare **ANR rate** and **stack** **fingerprints** for **that** **screen** / **process** **before vs after** (same **time window** length, watch for **seasonality**).  
- **Same device class / OS**: cheap segment in Play—**low-RAM** phones often **hurt** first.  
- **Trace again** on a **release** build (or **profileable** **release**): confirm **main** is **clean** on the **path** we fixed.

---

**2. Guard against regressions — what usually breaks**

| Change type | Regression risk | What I verify |
|-------------|------------------|---------------|
| **Moved work** to **background** | **Race** conditions, **stale** UI, **double** submit | **Unit** tests + **UI** test for **happy** / **error** / **fast** repeat taps |
| **New** **caching** | **Wrong** data shown, **memory** | **Integration** tests, **memory** **profiler** on **scroll** |
| **Dispatcher** / **scope** changes | **Leaks**, **cancel** bugs | **LeakCanary** (debug), **structured** **concurrency** review |
| **Baseline Profile** / **R8** tweaks | **Cold** **start** elsewhere | **Macrobenchmark** on **launcher** + **1–2** **critical** flows |

---

**3. Automated checks (repeatable proof)**  
- **Macrobenchmark** / **Microbenchmark** for **frame** **time** / **startup** on **CI** (or **nightly**) for **touched** **flows**.  
- **Espresso** / **Compose** **UI** tests for **regression** **paths** tied to the **change**.  
- **Unit** tests for **pure** **logic** moved out of the **UI** thread.

---

**4. Release strategy (catch issues before everyone gets them)**  
- **Internal** + **QA** builds with **logging** **switches** **off** by default in **prod**, **on** in **test**.  
- **Staged** **rollout** (for example **5% → 25% → 100%**) with **rollback** plan (**flag** off or **hotfix**).  
- **Dashboards**: **ANR** %, **crash-free** **sessions**, **latency** **p50/p95** for **key** APIs.

---

**5. “Other areas of the app” — how I keep scope sane**  
I make a **short** **blast radius** list: **shared** **modules**, **DI** **graph**, **global** **OkHttp** **interceptors**, **image** **loading** **defaults**. I run a **smoke** **suite** and **manual** **spot-checks** on those **entry** points even if they look **unrelated**.

---

**6. Trade-off**  
**Full** **perf** **CI** is **slow** and **flaky** on **emulators**—I **prioritize** **benchmarks** for **what** **changed**, not **every** **screen**.

**One-line close:** *“Baseline production ANRs and timings, validate with release-like traces and targeted benchmarks/tests, then roll out in stages with monitoring and a rollback lever—prove the fix on metrics, contain regressions by tracing blast radius.”*

---

### Question 5

**I'm curious about a different area now—tell me how you would design a secure networking layer when integrating a new RESTful backend using Retrofit and SSL pinning—specifically, how would you handle token-based authentication on the device?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I put **TLS** **pinning** and **token** **handling** in **OkHttp** (**CertificatePinner**, **interceptors**, **single-flight** **refresh**), store **tokens** in **encrypted** storage, and **never** log **secrets**.  
- **D — Detail:** _(your example)_ **DataStore** / **EncryptedSharedPreferences** + **401** **refresh** with a **mutex**—*(banking app login flow you built)*.  
- **D — Describe relevance:** **EBE** **banking** clients must resist **MITM** and **token** **theft**—this is **table-stakes** **security** on **Android**.

**Short version (about 30 seconds)**  
I build **one** **`OkHttpClient`** used by **Retrofit**: **`CertificatePinner`** pins the **server** **public keys** (**SPKI** **hashes**), with a **backup** **pin** for **rotation**. **Interceptors** add **`Authorization: Bearer <access>`** and handle **401** with a **safe**, **single** **refresh** flow (**mutex** / **one** **in-flight** **refresh**) so we **do not** **storm** the **token** **endpoint**. **Tokens** live in **encrypted** storage (**DataStore** with **encryption** or **`EncryptedSharedPreferences`**), **not** plain **SharedPreferences**; **refresh** tokens get **stricter** handling when the backend allows **separation**. **No** tokens in **logs** or **crash** reports; **release** builds use **redacted** **OkHttp** logging or **none**.

---

**1. Stack shape — Retrofit on pinned OkHttp**  
- **Retrofit** only needs the **API** **interfaces**; **all** **TLS** trust and **pins** sit in **`OkHttpClient`**.  
- **One** **client** per **environment** (**prod** / **staging**) so **pins** and **timeouts** do **not** **leak** **across** **hosts**.

---

**2. SSL pinning — what I configure**  
- **`CertificatePinner`** on **`OkHttp`**: pin **SPKI** **hashes** (not only **leaf** cert—plan for **chain** **changes**).  
- **Backup** **pin** in **config** so a **planned** **cert** **rotation** does **not** **brick** **every** **user** at **once**.  
- **Process** with **backend**/**DevOps**: know **rotation** **dates**, **test** **builds** **before** **pin** **updates** **ship**.

**Words you can explain:** **Pinning** = the app **only** trusts **specific** **keys** for **that** **host**, not **any** **CA-valid** cert—**stronger** than **default** **trust**, but **breaks** if pins are **wrong**.

---

**3. Token-based auth on the device — storage**

| Piece | Practical approach |
|-------|---------------------|
| **Access token** | **Short-lived**; store **encrypted**; load into **memory** while app **foreground** when possible. |
| **Refresh token** | **Longer-lived**; **tighter** storage (still **encrypted**); some teams use **Keystore**-backed **wrapping** for **extra** **defense**. |
| **Logout / revoke** | **Delete** **all** **tokens** from **storage** + **clear** **memory**; **cancel** **in-flight** **calls** if **needed**. |

I avoid **hard-coding** **secrets** in **APK**; **client** **IDs** may be **public** by design, but **tokens** are **never** **committed** to **git**.

---

**4. Interceptors — attach and refresh**  
- **`Interceptor`** (or **`Authenticator`** for **401**): **read** **access** **token** from a **small** **token** **repository** / **session** **store**, **attach** **`Authorization`**.  
- **401** **handling:** **refresh** **once**, **retry** **original** **request** **once**; if **refresh** **fails** → **force** **logout** / **login** **screen**.  
- **Concurrency:** **only** **one** **refresh** at a **time** (**mutex**, **`Mutex`** in **coroutines**, or **OkHttp** **Authenticator** **patterns**) so **parallel** **401s** do **not** **create** **many** **sessions**.

---

**5. Defense in depth (short)**  
- **Cleartext** **off** (`usesCleartextTraffic=false`) unless **strictly** **needed** in **debug**.  
- **Trust** **user** **certs** **only** in **debug** if **required** (**network** **security** **config**).  
- **Certificate** **transparency** / **modern** **TLS** are **mostly** **server** **side**, but I **choose** **libraries** and **minSdk** that **support** **good** **defaults**.

---

**6. Trade-offs**  
- **Pinning** **too** **aggressively** **without** **backup** **pins** → **outages** when **certs** **rotate**.  
- **Encrypted** **storage** still **loses** to **rooted** **devices**—**threat** **model** should **assume** **tokens** can be **read** on a **compromised** **phone**; **backend** **limits** **scope**, **lifetime**, and **revocation** matter **most**.

**One-line close:** *“Pin TLS with OkHttp CertificatePinner plus backup pins; centralize tokens in encrypted storage with a single-flight refresh interceptor; never log secrets—treat the client as untrusted and keep access tokens short.”*

---

### Question 6

**How would you handle error responses from the backend, such as retries or exponential backoff, to ensure resilience in your networking layer?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I **classify** errors, **retry** only **transient** failures with **bounded** **backoff** **+** **jitter**, **respect** **429**, and **never** **auto-retry** unsafe **POSTs** without **idempotency**—then **map** to **domain** errors for the **UI**.  
- **D — Detail:** _(your example)_ **Repository**-level **retry** policy; **verify** **payment** **status** on **unknown** **writes**—*(API behavior you handled)*.  
- **D — Describe relevance:** **Banking** **networking** must avoid **double** **charges** and **honest** **UX** under **flaky** **mobile** **networks**.

**Short version (about 30 seconds)**  
I **split** problems into **three** buckets: **transport** errors (**timeouts**, **no** **network**), **server** **errors** (**5xx**), and **client** **errors** (**4xx**). I **only** **retry** when it is **safe** and **likely** to **help**: **transient** **network** / **5xx**, with **caps** (**max** **attempts**, **max** **delay**). **Backoff** is **exponential** with **jitter** so **many** **phones** do **not** **hit** the **server** at the **same** **instant**. I **do not** **blindly** **retry** **non-idempotent** **writes** (**POST** **payments**) **without** **idempotency** **keys** or **clear** **server** **rules**. **429** respects **`Retry-After`** when **present**. At the **edge** of the **app**, I **map** errors to **stable** **domain** **types** so the **UI** shows **honest** **states**, not **raw** **HTTP** **codes**.

---

**1. First: classify the error (retry or stop)**

| Signal | Typical action |
|--------|----------------|
| **IOException** / **timeout** / **TLS** **handshake** | **Retry** with **backoff** (transient). |
| **5xx** | **Retry** with **backoff** (server **overload** / **bug**). |
| **401** | **Refresh** **token** / **re-login**—**not** “retry forever.” |
| **403** | Usually **no** **retry** (policy). |
| **404** | **No** **retry** (wrong **path** / **resource**). |
| **409** / **422** | **Business** **validation**—show **message**, **no** **retry** **loop**. |
| **429** | **Respect** **`Retry-After`**; **backoff** anyway. |

---

**2. Backoff — what “good” looks like**  
- **Exponential:** delay grows: for example **250ms → 500ms → 1s** (with a **ceiling**).  
- **Jitter:** add **random** **spread** (**full** **jitter** is a **common** **pattern**) so **retries** **don’t** **align** (**thundering** **herd**).  
- **Limits:** **max** **attempts** (for example **3**) and **max** **total** **wait**—then **fail** **cleanly** to the **UI**.

---

**3. Where the logic lives**  
- **OkHttp** gives **connection** **retries** for **some** **cases**—I **do not** rely on it **alone** for **application-level** **policy**.  
- **Repository** / **remote** **data** **source:** **one** **policy** **function** (or **small** **library** **wrapper**) so **Retrofit** **calls** share the **same** **rules**.  
- **Kotlin:** **`retry`** / **`retryWhen`** on **Flow**, or a **loop** with **`delay`** around **suspend** **calls**—always **structured** **concurrency** so **cancellation** still **works**.

---

**4. Idempotency — the banking rule**  
**GET** is **usually** **safe** to **repeat**. **POST** **that** **moves** **money** is **not** **safe** to **repeat** **unless** the **API** supports **idempotency** **keys** or **dedup** on the **server**. My **default**: **no** **auto-retry** **writes** **without** **that** **contract**; **show** **“unknown”** **state** and **verify** **status** **instead** of **double** **submit**.

---

**5. Mapping for the UI**  
**Network** **layer** returns **typed** **results** (**sealed** **classes** / **`Result`**) with **user-safe** **messages** and **telemetry** **codes** (**no** **PII** in **logs**). The **ViewModel** **does** **not** **parse** **Retrofit** **errors** **directly**—it **consumes** **domain** **failure** **types**.

---

**6. Optional: circuit breaker**  
If the **service** is **down** for **everyone**, **endless** **retries** **hurt** **battery** and **load**. A **simple** **breaker** (**open** after **many** **failures**, **half-open** **probe**) can be **worth** it for **high-traffic** **apps**—**team** **decision**.

---

**7. Trade-off**  
**Aggressive** **retries** **hide** **slow** **backends** and can **duplicate** **side** **effects**. **Policy** + **metrics** (**retry** **count**, **final** **outcome**) keep **resilience** **honest**.

**One-line close:** *“Retry only transient failures with bounded exponential backoff and jitter, respect 429 and idempotency rules, map errors to domain types for the UI—and never auto-retry unsafe writes without server support.”*

---

### Question 7

**Now I want to make sure we cover another area—how would you improve build and release reliability using Gradle and CI/CD, to speed up builds and enforce code quality for the Android team?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I speed up **Gradle** (**config** **cache**, **build** **cache**, **modules**), **pin** **toolchains**, and put **lint/tests** on **every** **PR** with **repeatable** **release** **pipelines** and **secrets** **outside** **git**.  
- **D — Detail:** _(your example)_ **GitHub** **Actions** / **Bitrise** with **Gradle** **cache**; **internal** **track** on **Play**—*(your CI setup)*.  
- **D — Describe relevance:** A **Senior** is expected to **multiply** the **team** with **fast**, **safe** **shipping**—**especially** in **regulated** **releases**.

**Short version (about 30 seconds)**  
On **Gradle** I turn on **safe** **speed** **features**: **configuration** **cache**, **parallel** **execution**, **build** **cache** (and **remote** **cache** if the **team** **can** **host** it), and **keep** **modules** **small** **enough** that **incremental** **work** **wins**. I **pin** the **Gradle** **Wrapper** and use a **version** **catalog** so **everyone** and **CI** use the **same** **dependencies**. On **CI/CD** I **cache** **Gradle** **home** and **wrapper**, run **`lint`**, **unit** **tests**, and **static** **analysis** on **every** **PR** (**fail** **fast**), and **build** **release** **artifacts** on **merge** with **clear** **signing** **secrets** in **CI** **stores**, not **in** **git**. **Release** **reliability** grows when **pipelines** are **repeatable**: **same** **commands** **locally** and on **servers**, **staged** **tracks** to **Play**, and **rollback** **rules**.

---

**1. Speed — Gradle levers that pay off**

| Lever | What it does |
|-------|----------------|
| **Configuration Cache** | **Skips** **re-running** **configuration** when **scripts** **did not** **change**. |
| **Build Cache** | **Reuses** **outputs** from **other** **builds** (local; **remote** **helps** **CI**). |
| **`org.gradle.parallel=true`** | **Modules** **build** **in** **parallel** on **multi-core** **CI**. |
| **Modularization** | **Smaller** **units** → **faster** **incremental** **compile** when **one** **feature** **changes**. |
| **KSP** instead of **kapt** where **possible** | **Less** **stub** **work**, **faster** **annotation** **processing**. |

I also **avoid** **giant** **`allprojects`** **scripts** **without** **structure**—**convention** **plugins** (**`build-logic`**) keep **build** **code** **testable** and **consistent**.

---

**2. Reliability — same build everywhere**  
- **Gradle** **Wrapper** (`./gradlew`) **committed**—**no** “works on my machine” **Gradle** **version**.  
- **JDK** **version** **fixed** in **CI** and **documented** for **local** (**toolchains** help).  
- **`gradle.properties`** **tuned** **once** for the **team**, **reviewed** like **product** **code**.

---

**3. CI/CD — what runs when**

| Stage | Typical gates |
|-------|----------------|
| **On PR** | **`assembleDebug`** or **`testDebugUnitTest`**, **Android** **Lint**, **Detekt** / **ktlint**, **(optional)** **small** **UI** **smoke** |
| **On main** | **Release** **bundle** **build**, **ProGuard**/**R8** **rules** **check**, **upload** to **internal** **track** |
| **Nightly** | **Heavier** **tests**, **Macrobenchmark** **sample**, **dependency** **updates** **scan** |

**Caching** on CI: **Gradle** **user** **home** + **dependencies**—**big** **time** **save** when **done** **right**.

---

**4. Code quality — enforce, don’t hope**  
- **Lint** + **Detekt**/**ktlint**: **warnings** **policy** (**zero** **new** **warnings** on **touched** **files** or **strict** **baseline** **over** **time**).  
- **Danger** / **bot** **comments** (optional): **surface** **diff** **size**, **missing** **tests**.  
- **Codeowners** for **critical** **modules** (optional).

---

**5. Release pipeline (short)**  
- **Secrets**: **Play** **signing** and **upload** **keys** in **CI** **secret** **store**, **not** **repo**.  
- **Tracks**: **internal** → **closed** → **production** with **staged** **rollout** where **possible**.  
- **Artifacts**: **AAB** **stored** with **version** **code** / **Git** **SHA** **tag** for **traceability**.

---

**6. Trade-off**  
**Remote** **cache** and **heavy** **CI** **need** **maintenance** and **cost**. I **start** with **local** **build** **cache** + **PR** **checks**, then **add** **remote** **cache** when **CI** **time** **hurts** **the** **team**.

**One-line close:** *“Tune Gradle for cache and parallelism, modularize for incremental builds, pin toolchain and dependencies, and run lint/tests/static analysis on every PR with cached CI jobs—then automate signed releases with secrets outside git and staged rollout.”*

---

### Question 8 (follow-up)

**What specific step would you take to reduce Gradle build times in a large Android project, focusing on one key bottleneck you've encountered in practice?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** The biggest win I have seen is **splitting** a **kapt-heavy** **monolith** and **moving** **Room** to **KSP**, **proved** with **Gradle** **Build** **Scans**.  
- **D — Detail:** _(your example)_ **Before/after** **annotation** **processing** time on a **one-line** **DAO** **change**—*(real module split)*.  
- **D — Describe relevance:** **Faster** **builds** mean **faster** **feedback** for **every** **Android** **dev** on **EBE**-sized **codebases**.

**Short version (about 30 seconds)**  
A **bottleneck** I have **seen** **often** is **heavy** **`kapt`** **work** (**Hilt**, **Room**, **Moshi**) **inside** a **very** **large** **single** **module**: **every** **small** **edit** **forced** **a** **big** **re-compile** and **re-annotation** **pass**. The **one** **step** I **took** that **moved** the **needle** was **splitting** **that** **monolith** into **smaller** **Gradle** **modules** (**features** + **core**/**data**) so **Gradle** could **skip** **unchanged** **modules**, and **migrating** **Room** **from** **`kapt`** **to** **`KSP`** **in** **the** **data** **module** **first**—**KSP** is **faster** and **fits** **incremental** **builds** **better**. I **proved** it with a **Gradle** **Build** **Scan** (**--scan**) **before/after** on a **typical** **edit** **flow**.

---

**1. The bottleneck (in plain words)**  
**`kapt`** **stubs** **and** **processors** **can** **dominate** **compile** **time** when **many** **files** **and** **processors** **live** **together**. In a **monolith** **`:app`**, **any** **change** **touches** **one** **giant** **compile** **unit**, so **incremental** **Kotlin** **compile** **helps** **less** than **you** **hope**.

---

**2. The specific step I’d name in an interview**  
**Modularize** **first**, **then** **replace** **`kapt`** **with** **`KSP`** **where** **supported** (**Room** is a **classic** **win**):

1. **Draw** **module** **boundaries**: **`:feature:*`**, **`:core:*`**, **`:data`** (names **vary**).  
2. **Move** **Room** **(+** **Retrofit** **mappers)** **into** **`:data`** **and** **switch** **Room** **to** **`KSP`**.  
3. **Keep** **Hilt** **modules** **small** and **close** **to** **the** **code** **they** **wire**—sometimes **Hilt** **aggregators** **still** **use** **`kapt`**, but **the** **Room** **slice** **stops** **paying** **kapt** **tax** **for** **every** **DAO** **change** **in** **the** **whole** **app** **module**.

This is **one** **concrete** **lever**: **smaller** **compile** **graphs** + **faster** **codegen** **for** **Room**.

---

**3. How I verify (not vibes)**  
- Run **`./gradlew :app:assembleDebug --scan`** (or **CI** **scan** **links**) and **compare** **“Annotation** **processing”** and **“Kotlin** **compile”** **time**.  
- **Repeat** **the** **same** **one-line** **change** **before** **and** **after** **the** **split**/ **KSP** **migration**.

---

**4. Trade-off**  
**Modules** **add** **coordination** **cost** (**API** **surface**, **dependency** **rules**). I **don’t** **split** **“just** **because**”—I **split** **when** **build** **scans** **show** **compile** **or** **kapt** **concentration** **in** **one** **place**.

**One-line close:** *“In practice I attack oversized kapt-heavy monoliths by modularizing so Gradle can skip work, then migrate Room to KSP in the data module—verified with Gradle Build Scans on a typical edit.”*

---

### Question 9

**Understood—moving to the final area, how would you approach collaborating with a cross-functional squad to deliver a complex UI using Jetpack components, ensuring both performance and accessibility standards are met?**

**Answer (simple English, enhanced)**

**ADD**  
- **A — Answer:** I align **early** on **states** and **a11y** with **product/design/QA**, then ship **Compose**/**Navigation**/**ViewModel** with **profiled** **performance** and **TalkBack**-ready **semantics**.  
- **D — Detail:** _(your example)_ **LazyColumn** **keys**, **Macrobenchmark** on **jank**, **large** **font** **QA**—*(complex screen you shipped)*.  
- **D — Describe relevance:** **EBE** **banking** UIs must be **fast** and **inclusive**—**accessibility** is often a **legal** and **brand** requirement, not a **nice-to-have**.

**Short version (about 30 seconds)**  
I **start** **early** with **product** **and** **design**: **one** **shared** **understanding** of **states** (**loading**, **empty**, **error**, **success**) and **edge** **cases**, **plus** **accessibility** **targets** (**TalkBack**, **font** **scale**, **touch** **size**) **before** **pixels** **freeze**. On **Android** I **build** with **Jetpack** **Compose** (or **Views** + **Material** if the **project** **requires**), **Navigation**, **ViewModel**, and **stable** **lists** (**`LazyColumn`** / **`RecyclerView`**). **Performance** is **baked** **in**: **stable** **parameters**, **keys** **for** **lists**, **avoid** **heavy** **work** **during** **composition**, **profile** **jank** with **Layout** **Inspector** / **Macrobenchmark**. **Accessibility** is **not** **last**: **`semantics`**, **labels**, **focus** **order**, **contrast** **checked** with **design**, and **QA** **runs** **real** **TalkBack** **passes** **before** **release**.

---

**1. Cross-functional rhythm — how the squad works together**

| Partner | What I align on early |
|---------|------------------------|
| **Product** | **User** **journeys**, **must-have** **vs** **later**, **definition** **of** **done** **for** **states**. |
| **Design** | **Responsive** **layouts**, **max** **text** **length**, **dark** **mode**, **motion** **budget** (too **much** **animation** **hurts** **perf** + **a11y**). |
| **Backend** | **Pagination**, **field** **shape**, **latency** **assumptions** for **skeleton** **UI** / **timeouts**. |
| **QA** | **Test** **matrix** (**small** **phone**, **large** **font**, **TalkBack**). |

**Ceremony** **light** **but** **real**: **mid-sprint** **design** **tech** **review** (“**can** **we** **build** **this** **with** **Compose** **without** **nested** **scroll** **hell**?”), **demo** **to** **design** **on** **device**.

---

**2. Jetpack building blocks for a complex UI**  
- **UI:** **Compose** + **Material** **3** **components** **where** **possible** (**Navigation** **Compose**, **scaffold**, **snackbar** **host**).  
- **State:** **ViewModel** + **`StateFlow`** **unidirectional** **data** **flow**—**fewer** **surprise** **recomposition** **bugs**.  
- **Lists:** **`LazyColumn`** **with** **`items`/`itemsIndexed`**, **stable** **keys**, **`remember`** **wisely**; **avoid** **allocating** **new** **lambdas** **everywhere** (**stable** **callbacks** **or** **references**).  
- **Images:** **Coil**/**Glide** **with** **sizing** **rules**—**large** **bitmaps** **are** **a** **classic** **jank** **source**.

---

**3. Performance — practical checklist**  
- **Measure** **first** (**jank** / **frame** **time**) **on** **a** **low** **device**.  
- **Keep** **composition** **cheap**: **`derivedStateOf`**, **`key`**, **split** **heavy** **child** **into** **smaller** **composables** **with** **narrow** **inputs**.  
- **Baseline** **Profiles** **for** **startup** **hot** **paths** **if** **the** **screen** **is** **critical**.  
- **Don’t** **fight** **the** **main** **thread**: **pagination** **and** **image** **decode** **size** **match** **the** **view** **size**.

---

**4. Accessibility — practical checklist**  
- **Semantics** **first** **class**: **merge** / **clear** **hierarchy**, **headings** **where** **needed**, **content** **descriptions** **for** **icons** **without** **visible** **text**.  
- **Touch** **targets** **≥** **48dp** **effective** **(or** **team** **standard**).  
- **Support** **font** **scaling** **and** **large** **display** **settings**—**no** **clipped** **text** **in** **production**.  
- **Keyboard**/**switch** **access** **on** **TV**/**tablet** **when** **relevant** (**focus** **order** **matches** **reading** **order**).  
- **Automate** **what** **we** **can**: **Lint** **a11y** **checks**, **manual** **TalkBack** **for** **each** **release** **slice**.

---

**5. Trade-off**  
**Perfect** **a11y** **+** **fancy** **animation** **for** **every** **screen** **costs** **time**. I **prioritize** **high-traffic** **flows** **and** **regulatory** **needs** **first**, **then** **iterate**.

**One-line close:** *“Align early on states and accessibility with product/design/QA; implement complex Jetpack UIs with stable state and list patterns; profile on real devices; ship semantics, labels, and TalkBack coverage as part of done—not as an afterthought.”*

---

## Quick reference — EBE context

_Use this section to jot company/product notes as you research (optional)._

- **What they do:** _(fill in)_
- **Stack / focus areas:** _(Kotlin, Compose, architecture, etc.)_
- **Why this role fits you:** _(one or two bullets)_

---

## After each session — self-check

- [ ] Did I tie answers to outcomes (performance, reliability, team impact)?
- [ ] Did I mention trade-offs and when I’d choose differently?
- [ ] Can I give one concrete example per behavioral/technical claim?
