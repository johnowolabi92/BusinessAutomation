# Playwright patterns

Use these patterns only with a permitted standalone Node Playwright runtime. A browser connector can expose a different API or prohibit filesystem, network, or evaluate operations; follow its documentation instead of copying these examples into it.

## Evidence-based interaction

The following names are illustrative. Replace them with names observed on the target page.

```js
import { chromium } from 'playwright';

const targetUrl = process.argv[2];
if (!targetUrl || !['http:', 'https:'].includes(new URL(targetUrl).protocol)) {
  throw new Error('Provide an HTTP(S) URL');
}
const browser = await chromium.launch({ headless: true });
try {
  const context = await browser.newContext();
  const page = await context.newPage();
  page.setDefaultTimeout(15_000);
  await page.goto(targetUrl, { waitUntil: 'domcontentloaded' });

  // Use labels and outcomes discovered from the actual page.
  const search = page.getByRole('textbox', { name: 'Search', exact: true });
  if (await search.count() !== 1) throw new Error('Search target is ambiguous');
  await search.fill('requested search text');
  await page.getByRole('button', { name: 'Search', exact: true }).click();
  await page.getByRole('heading', { name: 'Search results', exact: true })
    .waitFor({ state: 'visible' });
} finally {
  await browser.close();
}
```

Use `locator.selectOption()` for a native select. For a custom dropdown, click the observed control and then the observed option. Scope duplicate buttons with a row or dialog locator. Use `frameLocator()` only with an observed frame selector. Register a download or popup event wait before triggering its action. Avoid `force: true`, blind `nth()`, and arbitrary timeouts as fixes for uncertain state.

## Bounded traversal algorithm

1. Seed the queue with the provided URL at depth zero.
2. Pop one item; reject URLs outside the allowed origin/path and skip visited URLs.
3. Stop at the configured page limit; record the stop reason.
4. Navigate, verify the loaded page still belongs to scope, and extract the agreed fields.
5. Resolve observed anchors with `new URL(href, page.url())`; discard non-HTTP(S), fragments, and out-of-scope destinations.
6. Enqueue eligible unseen URLs only when below the depth limit. Track queued URLs too.
7. Record errors separately. On a rate limit, honor Retry-After within the task budget or stop with a partial-result report.

A generic link crawl does not prove all records were collected. Inspect pagination, filters, load-more buttons, and virtualized lists separately. Report the number of pages and rows read, duplicates removed, failures, and any limit reached.

## Useful verification

For extraction, compare a sample of records against visible rows and ensure requested fields exist. For a filter, confirm its displayed selection and that returned rows meet the filter. For a form, verify field values before an authorized submit and inspect the success or validation state afterward. For a reusable script, use a local fixture for ambiguous controls, empty results, duplicate pagination, and scope boundaries; a target-site run remains a separate check.
