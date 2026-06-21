# In-Placed Upgrade 完整流程

```mermaid
flowchart TD
    %% Styles

    Start([開始：Spring Boot 2.5 原地升級]) --> P0[Phase 0：Baseline 盤點<br/>不改 code，只記錄現況]

    P0 --> P1[Phase 1：建立 migration branch<br/>設定 rollback point]

    P1 --> P2[Phase 2：JDK 17 / Maven / IDE 環境準備]
    P2 --> JdkCheck{JDK 17+ 是否可用?}
    JdkCheck -- 否 --> FixJdk[修本機 / Maven / IDE / CI 的 JDK 設定]
    FixJdk --> JdkCheck

    JdkCheck -- 是 --> P3[Phase 3：OpenRewrite dryRun<br/>預覽 Spring Boot 3.5 自動修改]
    P3 --> DiffCheck{OpenRewrite diff 是否合理?}
    DiffCheck -- 否 --> TuneRecipe[調整 recipe / 排除不合理修改]
    TuneRecipe --> P3

    DiffCheck -- 是 --> P4[Phase 4：OpenRewrite run<br/>套用 Spring Boot 3.5 migration]
    P4 --> CommitRewrite[Commit 1：只提交 OpenRewrite 自動修改]

    CommitRewrite --> P5[Phase 5：部署模型切換<br/>WAR/WildFly → Spring Boot executable JAR]
    P5 --> CommitDeploy[Commit 2：只提交部署模型切換]

    CommitDeploy --> P6[Phase 6：修 dependency / main compile error]
    P6 --> CompileCheck{main code 是否 compile 成功?}
    CompileCheck -- 否 --> FixCompile[分類修復：jakarta / dependency / API removed / module conflict]
    FixCompile --> CompileCheck

    CompileCheck -- 是 --> P7[Phase 7：修 test compile error]
    P7 --> TestCompileCheck{test code 是否 compile 成功?}
    TestCompileCheck -- 否 --> FixTestCompile[修 JUnit / Mockito / MockMvc / Spring Test 問題]
    FixTestCompile --> TestCompileCheck

    TestCompileCheck -- 是 --> P8[Phase 8：跑 unit / integration test]
    P8 --> TestCheck{測試是否通過?}
    TestCheck -- 否 --> FixTest[分類修復：Security / JPA / Config / Controller / Service]
    FixTest --> TestCheck

    TestCheck -- 是 --> P9[Phase 9：本機 executable jar 啟動驗證]
    P9 --> RuntimeCheck{java -jar 是否啟動成功?}
    RuntimeCheck -- 否 --> FixRuntime[修 datasource / profile / logging / port / bean 問題]
    FixRuntime --> RuntimeCheck

    RuntimeCheck -- 是 --> P10[Phase 10：核心 API 行為比對]
    P10 --> ApiCheck{新舊 API 行為是否一致?}
    ApiCheck -- 否 --> FixApi[修 Controller / Service / Repository / SQL / DTO 差異]
    FixApi --> P10

    ApiCheck -- 是 --> P11[Phase 11：部署環境驗證<br/>確認正式啟動方式與設定注入]

    P11 --> P12[Phase 12：清理 legacy 設定與文件化]
    P12 --> End([完成：Spring Boot 3 In-place Upgrade])
```

# Phase 0：Baseline 盤點，不改 code

這一步最重要。你現在不要一開始就改 Spring Boot 版本。

你要先建立「舊系統原本的狀態」，否則 migration 後你會分不清楚：

```
這個錯是原本就存在？
還是我升級造成的？
```

你要記錄：

```
1. 目前 Java 版本
2. 目前 Maven 版本
3. 目前 Spring Boot 版本
4. 目前 packaging 是 war 還是 jar
5. 目前是否依賴 WildFly
6. 目前哪些 module 可以 build
7. 目前哪些 test 原本就失敗
8. 目前本機能不能啟動
9. 目前核心 API 的 request / response
10. 目前 datasource / profile / properties 如何注入
```

可以先跑：

```bash
java -version
mvn -version
mvn clean test
```

產出文件：

```
baseline-report.md
api-baseline.md
known-issues.md
environment-setup.md
```

這一步的完成標準不是「沒有錯」，而是：

> **你知道舊系統在 migration 前到底是什麼狀態。**
> 

---

# Phase 1：建立 migration branch

不要在主線或原本開發 branch 上直接改。

建議 branch：

```bash
git checkout -b migration/springboot-3-inplace
```

這條 branch 的原則是：

```
1. 每一階段單獨 commit
2. OpenRewrite 的 diff 和人工修正分開
3. 部署模型切換和 compile fix 分開
4. 每次 commit 都能說明「這包修改的目的」
```

建議 commit 結構：

```bash
git commit -m "docs: add Spring Boot 2.5 migration baseline"
git commit -m "chore: apply OpenRewrite Spring Boot 3.5 migration recipes"
git commit -m "build: switch from WildFly WAR deployment to Spring Boot executable jar"
git commit -m "fix: resolve compilation errors after Spring Boot 3 migration"
git commit -m "test: update tests for Spring Boot 3 compatibility"
git commit -m "docs: add Spring Boot 3 migration verification report"
```

---

# Phase 2：先準備 JDK 17，不要急著改 Spring Boot

Spring Boot 3 要求 Java 17+，所以你不能在 JDK 8 環境下期待 Spring Boot 3 正常 build。(GitHub)

你要先確認：

```bash
java -version
mvn -version
```

你需要看到 Maven 實際使用的是 JDK 17 或以上。

這裡要注意：

```
IDE 設定的 JDK
命令列 java -version
Maven 使用的 JDK
公司開發機可用的 JDK
CI/CD 可用的 JDK
```

這幾個可能不是同一個。

完成標準：

```
[ ] 本機 JDK 17+ 可用
[ ] Maven 使用 JDK 17+
[ ] IDE project SDK 設為 JDK 17+
[ ] 如果有 CI，CI 也能使用 JDK 17+
```

---

# Phase 3：OpenRewrite dryRun，先看它會改什麼

這一步是「預覽」，不是正式修改。

OpenRewrite 的 Spring Boot 3.5 recipe 會修改 build files，並處理 deprecated/preferred APIs；官方也有 `UpgradeSpringBoot_3_5` recipe。(OpenRewrite Docs)

建議先跑 dry run：

```powershell
mvn -U org.openrewrite.maven:rewrite-maven-plugin:dryRun `
  "-Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:RELEASE" `
  "-Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_5"
```

你要檢查它改了什麼：

```
1. pom.xml 是否升級到 Spring Boot 3.5.x
2. java.version 是否變成 17+
3. javax 是否改成 jakarta
4. application.properties / yml 是否有被改
5. 有沒有改到你不希望它改的地方
6. diff 是否過大到不可 review
```

如果 diff 太誇張，不要直接 run。

先調整 recipe 或縮小範圍。

---

# Phase 4：OpenRewrite run，正式套用自動修改

正式套用：

```powershell
mvn -U org.openrewrite.maven:rewrite-maven-plugin:run `
  "-Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:RELEASE" `
  "-Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_5"
```

套用後立刻：

```bash
git diff
```

這個 commit 只放 OpenRewrite 自動產生的修改：

```bash
git add .
git commit -m "chore: apply OpenRewrite Spring Boot 3.5 migration recipes"
```

不要把你自己的修正混進去。

原因是 reviewer 需要分得出：

```
這些是工具自動改的
那些是你人工判斷後改的
```

---

# Phase 5：部署模型切換：WAR/WildFly → Spring Boot executable JAR

這是你現在最新補充的關鍵。

因為未來不再用 WildFly，所以你不需要再維持傳統 WAR 部署路線。你要把部署模型改成 Spring Boot standalone executable jar。Spring Boot Maven plugin 可以產生包含相依套件、可用 `java -jar` 執行的 executable archive。(Home)

## 你要改的地方

### 1. `pom.xml` packaging

舊模式可能是：

```xml
<packaging>war</packaging>
```

改成：

```xml
<packaging>jar</packaging>
```

或直接移除，因為 Maven 預設是 jar。

---

### 2. 移除 Tomcat provided scope

舊 WAR 模式可能有：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

改成 standalone jar 後，通常不要再 provided。

你可以只保留：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

讓 embedded Tomcat 進入 runtime classpath。

---

### 3. 確認 Spring Boot Maven Plugin

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

---

### 4. 簡化啟動類

如果現在是：

```java
@SpringBootApplication
public class StationaryApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(StationaryApplication.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(StationaryApplication.class, args);
    }
}
```

確定不再用 WAR 後，可以簡化成：

```java
@SpringBootApplication
public class StationaryApplication {

    public static void main(String[] args) {
        SpringApplication.run(StationaryApplication.class, args);
    }
}
```

這不是一定必要，但會讓架構意圖更清楚。

這一步可以單獨 commit：

```bash
git add .
git commit -m "build: switch from WildFly WAR deployment to Spring Boot executable jar"
```

---

# Phase 6：修 dependency / main compile error

接下來才開始修編譯錯誤。

先跑：

```bash
mvn clean compile -DskipTests
```

你會遇到的錯誤大致分成：

```
A. javax / jakarta 漏網之魚
B. 舊 dependency 不支援 Spring Boot 3
C. Spring API 被移除
D. Hibernate / JPA 變更
E. Servlet API 變更
F. Maven module dependency conflict
G. annotation processor 問題
H. 舊公司內部 jar 不相容
```

你的修法不要看到錯就亂改，要分類處理。

建議順序：

```
1. 先修 pom dependency conflict
2. 再修 jakarta import 問題
3. 再修 Spring API removed 問題
4. 再修 JPA / Hibernate 問題
5. 最後修 module 之間引用問題
```

這一步完成標準：

```bash
mvn clean compile -DskipTests
```

成功。

---

# Phase 7：修 test compile error

這一步不是修測試結果，而是先讓測試程式碼本身可以編譯。

跑：

```bash
mvn test-compile
```

常見問題：

```
1. JUnit 4 / JUnit 5 混用
2. Mockito API 變更
3. MockMvc 設定問題
4. SpringBootTest context 載入失敗
5. javax validation import 沒改乾淨
6. 舊測試綁死 Spring Boot 2 behavior
```

這一步完成標準：

```
main code compile 成功
test code compile 成功
```

還不要求所有測試通過。

---

# Phase 8：跑 unit / integration test

現在才開始跑：

```bash
mvn test
```

如果有 integration test：

```bash
mvn verify
```

這時候你要把 test failure 分類，不要混在一起：

| 類型 | 可能原因 | 處理方式 |
| --- | --- | --- |
| Context load failed | Spring config / bean / profile 問題 | 修設定或 bean |
| Repository test failed | Hibernate 6 / SQL / mapping 問題 | 修 JPA mapping 或 query |
| Controller test failed | request / response / validation 行為改變 | 檢查 API contract |
| Security test failed | Spring Security 6 行為變更 | 修 security config |
| Mock test failed | Mockito / test setup 問題 | 修測試寫法 |

完成標準：

```
核心 unit test 通過
核心 integration test 通過
舊測試若不再有效，需要標記原因，不是直接刪掉
```

---

# Phase 9：本機 executable jar 啟動驗證

這一步是你的新部署模型驗證。

先打包：

```bash
mvn clean package
```

然後啟動：

```bash
java -jar target/your-app-name.jar
```

你要確認：

```
[ ] jar 可以啟動
[ ] port 正確
[ ] context path 正確
[ ] profile 正確
[ ] datasource 正確
[ ] logging 正確
[ ] health check 可打
[ ] DB 連線成功
[ ] 啟動時沒有 hidden warning / fallback config
```

這一步很關鍵，因為你已經不再用 WildFly。

所以未來最重要的驗證不再是：

```
能不能 deploy 到 WildFly
```

而是：

```
能不能用 java -jar 在目標環境啟動
```

---

# Phase 10：核心 API 行為比對

這一步取代你原本 Strangler flow 裡的「一支 API 一支 API 搬遷」。

In-place upgrade 不需要搬 API，但還是需要逐支驗證 API 行為。

你要建立一張 API comparison table：

| API | 舊版狀態 | 新版狀態 | 差異 | 結論 |
| --- | --- | --- | --- | --- |
| GET /api/items | 200 | 200 | response 一致 | OK |
| POST /api/order | 200 | 500 | validation 失敗 | Fix |
| GET /api/report | 200 | 200 | 欄位順序不同 | Accept |
| POST /api/upload | 200 | 413 | upload size 設定不同 | Fix |

核心做法：

```
同一組 request
舊版 Spring Boot 2.5 response
新版 Spring Boot 3.5 response
逐一比對
```

你可以先挑：

```
1. 查詢型 API
2. 影響小 API
3. 常用 API
4. 權限簡單 API
5. DB join 少的 API
```

再處理：

```
1. 上傳 API
2. 批次 API
3. 權限複雜 API
4. 交易 / 寫入 API
5. 報表 / 複雜 SQL API
```

注意：這裡不是搬遷 API，而是**驗證 API**。

這是從 Strangler 轉 In-place 後最重要的概念差異：

```
Strangler：
一支一支搬 API

In-place：
一支一支驗證 API
```

---

# Phase 11：正式部署環境驗證

因為未來是 Spring Boot 3 executable jar，所以你要確認正式環境怎麼啟動：

```bash
java -jar app.jar --spring.profiles.active=prod
```

或如果公司用 service/container：

```
systemd
Docker
Kubernetes
Jenkins pipeline
公司內部部署平台
```

你要確認：

```
[ ] JDK 17+ installed
[ ] 啟動命令確認
[ ] profile 注入方式確認
[ ] DB password / secret 注入方式確認
[ ] log path 確認
[ ] port / context path 確認
[ ] health check endpoint 確認
[ ] rollback 機制確認
```

這一步和 coding 沒有直接關係，但它決定 migration 是否真的能上線。

---

# Phase 12：清理 legacy 設定與文件化

最後才清理。

不要一開始就大清理，因為你會不知道錯誤是不是清理造成的。

最後可以清：

```
1. WildFly deployment descriptor
2. 舊 WAR profile
3. provided Tomcat 設定
4. 無用 properties
5. 無用 dependency
6. 無用 import
7. 不再使用的 legacy config
8. 過時文件
```

產出文件：

```
migration-report.md
deployment-guide-springboot3.md
api-verification-report.md
known-risk.md
rollback-plan.md
```

---

# API Counting Flow

```mermaid
1. 保留目前 API List 工作
2. 保留 Runtime Flowchart 工作
3. 新增 API Behavior Baseline 欄位
4. 對每支 API 標 P0 / P1 / P2
5. P0 API 完整記錄 request / response / error case
6. P1 API 記錄主要成功 response
7. P2 API 先只記 metadata 和測試狀態
8. 把現有測試類別對應到每支 API
9. migration 後用同一份表格逐支驗證
```

---

```
請從目前專案中盤點所有 REST API endpoint。

要求：
1. 只根據實際程式碼，不要猜測。
2. 必須解析 class-level @RequestMapping 與 method-level mapping 的完整組合。
3. 每個 API 需要列出：
   - HTTP Method
   - Full Path
   - Controller file
   - Controller method
   - Request DTO / parameters
   - Response type
   - 呼叫的 Service method
   - 呼叫的 DAO / Repository / Mapper
   - 可能讀取的 table
   - 可能寫入的 table
   - 是否有副作用，例如 send mail、batch、status update、external call
   - Risk Level: Low / Medium / High
   - Confidence: High / Medium / Low
   - Evidence: 檔案路徑 + method 名稱

4. 如果無法從程式碼確認 table 或 side effect，請標記為 Unknown，不要自行推測。
5. 請把 GET 但可能有副作用的 API 特別標出。
6. 請輸出成 Markdown table。
```

```
API:
POST /api/stationary/{id}/submit

盤點路徑:
1. Controller:
   StationaryController.submit()
   Evidence: @PostMapping("/{id}/submit")

2. Service:
   呼叫 stationaryService.submit(id)

3. DAO / Mapper:
   呼叫 stationaryMapper.findById(id)
   呼叫 stationaryMapper.updateStatus(request)

4. SQL / Table:
   SELECT FROM STATIONARY_REQUEST
   UPDATE STATIONARY_REQUEST SET STATUS = ...

5. Side Effect:
   是否寄信: 未確認
   是否簽核: 呼叫 workflowService，需人工確認

6. Risk:
   High

7. Confidence:
   Medium

8. Manual Check:
   需要確認 workflowService 是否真的觸發簽核流程
```

---

# 1. 測試都通過，還需要記錄 API 回傳結果嗎？

**需要，但可以有選擇地記錄。**

目前舊系統測試都通過，這是很好的 baseline，但它不一定等於 API 行為已經完整被保護。

原因是測試可能只驗證：

```
Service logic
Repository logic
Controller 某些狀態碼
MockMvc 部分結果
DAO 查詢結果
```

但不一定完整保證：

```
實際 HTTP status code
實際 JSON 欄位名稱
實際 response wrapper
實際錯誤格式
實際 header
實際 validation error
實際權限行為
實際 DB 資料下的 response
```

所以測試通過代表：

> **舊系統內部測試目前健康。**
> 

但 API baseline 代表：

> **舊系統對外行為目前長什麼樣子。**
> 

```mermaid
flowchart TD
    A[每支 API] --> B{是否有對應測試?}
    B -- 否 --> C[需要記錄 baseline request/response]
    B -- 是 --> D{測試是否覆蓋 HTTP status + response body?}
    D -- 否 --> E[需要補 baseline 或補測試]
    D -- 是 --> F{是否為 P0 核心 API?}
    F -- 是 --> G[仍建議保存一組代表性 response]
    F -- 否 --> H[可只記錄測試名稱與測試通過狀態]
```

---

# 2. 你不需要記錄「所有 API 的所有 response」，應該分層記錄。

## 必須記錄的 API

```
1. 核心業務 API
2. 前端常用 API
3. 有 DB 查詢 / JOIN 的 API
4. 有寫入、更新、刪除的 API
5. 有權限判斷的 API
6. 有檔案上傳 / 匯出 / 報表的 API
7. 之前容易出錯的 API
8. migration 後容易受 Spring Boot 3 影響的 API
```

```
核心 API 完整記錄
一般 API 簡要記錄
低風險 API 只標記測試覆蓋與狀態
```

## 可以延後記錄的 API

```
1. health check
2. 單純靜態查詢
3. 很少用的內部 API
4. 測試已經完整覆蓋的小 API
5. 不再使用或待廢棄 API
```

---

# 3. 建議你在 API List 裡新增這些欄位

你現在的 API List 建議變成這種格式：

| 欄位 | 說明 |
| --- | --- |
| API ID | 方便追蹤，例如 API-001 |
| Method | GET / POST / PUT / DELETE |
| Endpoint | `/api/items` |
| Controller | 對應 Controller method |
| Service | 主要 service |
| DAO / Repository | 主要資料來源 |
| Runtime Flowchart | 對應圖檔或 mermaid |
| Existing Test | 目前對應的測試類別 |
| Test Status | 舊系統測試是否通過 |
| Priority | P0 / P1 / P2 |
| Baseline Request | 舊系統測試 request |
| Baseline Response | 舊系統回傳結果 |
| Side Effect | 是否寫 DB / 改狀態 |
| Migration Risk | Low / Medium / High |
| Spring Boot 3 Risk | Jakarta / Security / JPA / Validation / Config |
| 新版驗證結果 | 後面 migration 後填 |

這樣你不是單純列 API，而是在建立 migration 控制表。

---

# 4. 每一支 API 需要記錄什麼？

對核心 API，至少記錄這些：

```
1. Request URL
2. HTTP method
3. Query params
4. Request body
5. Auth / session / token 條件
6. DB 前置資料條件
7. HTTP status code
8. Response body
9. 重要 headers
10. 錯誤情境 response
11. 是否有 DB side effect
12. 對應測試類別
```

例如：

```
API ID: API-012
Method: GET
Endpoint: /api/stationary/items?category=pen
Controller: StationaryController#getItems
Service: StationaryService#getItems
DAO: StationaryDao#findItems
Existing Test: StationaryControllerTest#getItems_shouldReturnItems
Test Status: Passed
Priority: P0
Migration Risk: Medium
Reason: 涉及 DB 查詢與 response DTO
```

Baseline Request：

```
GET /api/stationary/items?category=pen
Accept: application/json
```

Baseline Response：

```
{
  "success":true,
  "data": [
    {
      "id":1,
      "name":"ball pen",
      "category":"pen"
    }
  ],
  "message":null
}
```

Migration 後你要比對的是：

```
status code 是否一致
response wrapper 是否一致
欄位名稱是否一致
欄位型別是否一致
array/object 結構是否一致
error format 是否一致
```

---

# 5. 舊測試通過的情況下，你應該怎麼做？

你現在其實有一個很好的起點。你的 Phase 0 可以這樣定義：

## Phase 0A：測試 baseline

先記錄：

```
mvn clean test
```

產出：

```
目前所有測試通過
測試總數
各 module 測試狀態
surefire report / test report
```

你可以在文件寫：

```
Baseline Test Status:
- Date: YYYY-MM-DD
- Branch: xxx
- Java version: 8
- Spring Boot version: 2.5.x
- Command: mvn clean test
- Result: Passed
- Total tests: xxx
```

這代表舊系統本身是健康的。

---

## Phase 0B：API behavior baseline

接著針對 P0 / P1 API 記錄實際回傳。

不是因為不相信測試，而是因為你要建立：

```
migration 前後的外部行為比對依據
```

這個東西在 Spring Boot 3 migration 後非常有用。

---

# 6. 最佳做法：不要只手動記錄，最好可重跑

如果可以，建議你把 API baseline 做成可重跑的資料。

例如資料夾結構：

```
migration-baseline/
  api/
    API-001-get-items/
      request.md
      response-200.json
      response-error-400.json
      notes.md
    API-002-create-order/
      request.json
      response-200.json
      db-before.md
      db-after.md
      notes.md
  test-report/
    baseline-test-result.md
  api-list.md
```

如果你用 curl，可以這樣：

```
curl-s-D headers.txt \
-o response.json \
"http://localhost:8080/api/stationary/items?category=pen"
```

如果 response 是 JSON，可以格式化後保存：

```
jq-S . response.json > response.normalized.json
```

Windows PowerShell 可以先用：

```
curl.exe-s-Dheaders.txt-oresponse.json"http://localhost:8080/api/stationary/items?category=pen"
```

重點是留下：

```
request
response
headers
前置條件
```

這樣 migration 後可以再打一遍同樣 request。

---

# 7. 你應該如何排序 API baseline？

我建議你用三層。

## P0：一定要記錄完整 response

```
核心查詢 API
核心寫入 API
前端主流程 API
會影響資料正確性的 API
報表 / 匯出 API
權限相關 API
```

P0 要記錄：

```
正常 case
錯誤 case
空資料 case
權限不足 case
DB side effect
```

---

## P1：記錄主要成功 response

```
一般查詢 API
常用但風險中等 API
資料轉換較單純 API
```

P1 至少記錄：

```
正常 request
正常 response
status code
對應測試
```

---

## P2：只記錄 API metadata 與測試覆蓋

```
低風險 API
健康檢查 API
不常用 API
未來可能廢棄 API
```

P2 可以先不完整保存 response，只標記：

```
測試已覆蓋
目前通過
低風險
migration 後抽樣驗證
```

---

# 8. 你現在的工作應該調整成這樣

```mermaid
flowchart TD
    A[盤點 API List] --> B[標記 API Priority<br/>P0 / P1 / P2]
    B --> C[整理 Runtime Flowchart]
    C --> D[對應現有 Test Class]
    D --> E{是否為 P0 / P1 API?}
    E -- 否 --> F[只記錄 metadata + test status]
    E -- 是 --> G[建立 Baseline Request]
    G --> H[執行舊系統 API]
    H --> I[保存 Baseline Response]
    I --> J[記錄 DB 前置條件與 side effect]
    J --> K[加入 Migration Tracking Table]
```

---

# 9. 最重要的觀念

你現在不只是「看懂 API」。

你是在建立：

```
Spring Boot 3 migration 的驗收標準
```

也就是說，未來你升級後，不應該只是問：

```
能不能 compile？
```

而是要問：

```
這支 API 在 migration 後，是否仍然符合舊系統的行為？
```

如果你現在沒有記錄舊系統行為，migration 後就會變成憑印象判斷。

---

# How to Co-Develop with AI?

```mermaid
flowchart TD
    A[develop / main<br/>目前穩定版本] --> B[建立 migration/springboot3 branch]

    B --> C[在 migration branch 內小步 commit]
    C --> C1[commit: pom / JDK / dependency]
    C --> C2[commit: javax to jakarta]
    C --> C3[commit: security config]
    C --> C4[commit: JPA / Hibernate fix]
    C --> C5[commit: test fix]

    C1 --> D[中間狀態可能 compile fail]
    C2 --> D
    C3 --> D

    D --> E{是否已經形成可驗證版本?}

    E -- 否 --> C

    E -- 是 --> F[執行 mvn clean test]
    F --> G{CI/CD 是否通過?}

    G -- 否 --> C

    G -- 是 --> H[開 PR / merge 回 develop]
```

### Push Requirement

```
# 0. 建立專用分支
git switch -c migration/javax-to-jakarta

# 1. 確認工作區乾淨
git status

# 2. 把 AI 給的 diff 存成 patch
notepad migration.patch

# 3. 先看 patch 大概要改什麼
git apply --stat migration.patch

# 4. 檢查 patch 是否能套用
git apply --check migration.patch

# 5. 套用 patch
git apply migration.patch

# 6. 查看實際修改
git diff

# 7. 執行測試
mvn clean test

# 8. 確認要 commit 的檔案
git status

# 9. 加入暫存區
git add .

# 10. 再看一次 staged diff
git diff --cached

# 11. commit
git commit -m "Migrate javax imports to jakarta"
```

---
