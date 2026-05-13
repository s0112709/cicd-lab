# 314551089 CI/CD 作業報告

## CI Pipeline 說明

本次作業在專案中新增 `.github/workflows/ci_314551089.yaml`，讓 GitHub
Actions 在每次 push 到任一分支時自動執行 CI。Pipeline 使用 Node.js 22，
先透過 `npm ci` 安裝與 `package-lock.json` 完全一致的相依套件，再依序執行
TypeScript typecheck、Prettier check 與 Vitest 測試。

Pipeline 設計策略如下：

1. `TypeScript typecheck`：執行 `npm run typecheck`，使用 `tsc --noEmit` 檢查
   TypeScript 型別，避免型別錯誤進入主程式。
2. `Prettier check`：執行 `npm run format:check`，確認程式碼與設定檔格式符合
   Prettier 規範。
3. `Run tests and create JUnit report`：執行 Vitest，並同時產生
   `reports/vitest-junit.xml`。
4. `Publish test result summary`：讀取 JUnit XML，將測試總數、通過數、失敗數、
   skipped 數與執行時間寫入 `$GITHUB_STEP_SUMMARY`，因此可直接在 GitHub
   Actions 執行結果頁面看到測試摘要。
5. `Upload JUnit test report`：將 JUnit XML 作為 artifact 上傳，方便下載與檢查
   詳細測試紀錄。
6. `Fail when tests fail`：因為測試步驟需要先保留機會產生測試摘要，所以測試指令
   會先記錄 exit code；若 exit code 不為 `0`，最後再明確讓 job 失敗。

任一檢查失敗時，CI job 都會顯示失敗：

- TypeScript typecheck 或 Prettier check 失敗時，對應 step 會直接失敗並停止後續
  流程。
- Test 失敗時，workflow 會先產生測試摘要與 artifact，再由 `Fail when tests fail`
  step 讓 pipeline 顯示失敗。

## `.github/workflows/ci_314551089.yaml` 主要內容

```yaml
name: CI 314551089

on:
  push:
    branches:
      - '**'

permissions:
  contents: read

concurrency:
  group: ci-314551089-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    name: Typecheck, format, and test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: TypeScript typecheck
        run: npm run typecheck

      - name: Prettier check
        run: npm run format:check

      - name: Run tests and create JUnit report
        id: tests
        run: |
          mkdir -p reports
          set +e
          npm test -- --reporter=default --reporter=junit --outputFile.junit=reports/vitest-junit.xml
          status=$?
          echo "exit_code=$status" >> "$GITHUB_OUTPUT"
          exit 0

      - name: Publish test result summary
        if: always()
        run: |
          node <<'NODE'
          const fs = require('node:fs');

          const reportPath = 'reports/vitest-junit.xml';
          const summaryPath = process.env.GITHUB_STEP_SUMMARY;

          if (!summaryPath) {
            throw new Error('GITHUB_STEP_SUMMARY is not available.');
          }

          if (!fs.existsSync(reportPath)) {
            fs.appendFileSync(
              summaryPath,
              [
                '## Vitest Test Results',
                '',
                'JUnit report was not generated. The test command may not have started.',
                ''
              ].join('\n')
            );
            process.exit(0);
          }

          const xml = fs.readFileSync(reportPath, 'utf8');
          const suiteMatches = [...xml.matchAll(/<testsuite\b([^>]*)>/g)];

          const totals = suiteMatches.reduce(
            (acc, match) => {
              const attrs = Object.fromEntries(
                [...match[1].matchAll(/\s([a-zA-Z_:.-]+)="([^"]*)"/g)].map(([, key, value]) => [
                  key,
                  value
                ])
              );

              acc.tests += Number(attrs.tests || 0);
              acc.failures += Number(attrs.failures || 0);
              acc.errors += Number(attrs.errors || 0);
              acc.skipped += Number(attrs.skipped || 0);
              acc.time += Number(attrs.time || 0);
              return acc;
            },
            { tests: 0, failures: 0, errors: 0, skipped: 0, time: 0 }
          );

          const passed = totals.tests - totals.failures - totals.errors - totals.skipped;
          const status =
            totals.failures === 0 && totals.errors === 0 ? 'Passed' : 'Failed';

          const markdown = [
            '## Vitest Test Results',
            '',
            `| Status | Total | Passed | Failed | Errors | Skipped | Time |`,
            `| --- | ---: | ---: | ---: | ---: | ---: | ---: |`,
            `| ${status} | ${totals.tests} | ${passed} | ${totals.failures} | ${totals.errors} | ${totals.skipped} | ${totals.time.toFixed(2)}s |`,
            '',
            `JUnit report: \`${reportPath}\``,
            ''
          ].join('\n');

          fs.appendFileSync(summaryPath, markdown);
          NODE

      - name: Upload JUnit test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: vitest-junit-report
          path: reports/vitest-junit.xml
          if-no-files-found: warn

      - name: Fail when tests fail
        if: steps.tests.outputs.exit_code != '0'
        run: exit 1
```

## CI 執行結果截圖

成功執行截圖放置位置：

![GitHub Actions 成功執行截圖](./images/actions-success.png)

截圖中需要呈現 GitHub Actions workflow `CI 314551089` 成功完成，並可看到
`TypeScript typecheck`、`Prettier check`、`Run tests and create JUnit report`、
`Publish test result summary` 等 steps 執行成功。

GitHub Actions 測試結果摘要截圖放置位置：

![GitHub Actions 測試結果摘要截圖](./images/actions-test-summary.png)

截圖中需要呈現 `Vitest Test Results` 表格，包含 Total、Passed、Failed、Errors、
Skipped 與 Time 欄位。

## 失敗案例說明

本次失敗案例可使用「測試失敗」來驗證 pipeline 行為。故意將
`test/app.test.ts` 中首頁訊息的預期值改錯，例如：

```ts
expect(response.json().message).toBe('wrong message');
```

修改後 push 到 GitHub，Vitest 會因為實際回傳值仍為
`CI/CD Lab Fastify app is running` 而失敗。此時 workflow 仍會執行
`Publish test result summary` 並顯示測試失敗摘要，最後
`Fail when tests fail` step 會使整個 pipeline 顯示 failed。

失敗執行截圖放置位置：

![GitHub Actions 失敗執行截圖](./images/actions-failed.png)

錯誤原因：測試中的 expected message 與 Fastify app 實際回傳的 message 不一致。

修正方式：將測試預期值改回正確字串：

```ts
expect(response.json().message).toBe('CI/CD Lab Fastify app is running');
```

修正後重新 push，GitHub Actions 會重新執行，所有檢查通過後 pipeline 會顯示
success。

## 使用工具與策略

- GitHub Actions：負責在 push 時自動啟動 CI pipeline。
- `actions/checkout@v4`：取出 repository 原始碼。
- `actions/setup-node@v4`：安裝 Node.js 22 並啟用 npm cache。
- `npm ci`：依照 lockfile 安裝固定版本相依套件，確保 CI 環境可重現。
- TypeScript `tsc --noEmit`：只做型別檢查，不產生編譯輸出。
- Prettier `--check`：檢查格式，不在 CI 中自動修改檔案。
- Vitest：執行測試並輸出 JUnit XML。
- `$GITHUB_STEP_SUMMARY`：將測試結果整理成 Markdown 表格，直接顯示於 GitHub
  Actions 結果頁面。
- `actions/upload-artifact@v4`：保留 JUnit XML 測試報告，便於事後查閱。
