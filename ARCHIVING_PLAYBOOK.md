# Playbook: Archiving a Job Posting for Citation

Instructions for Claude Code. To use: start a session and say
**"Follow ~/job-posting-archives/ARCHIVING_PLAYBOOK.md to archive this posting: \<URL\>"**
(optionally pointing at a draft HTML file to verify). Everything below is automatable
except one manual step noted at the end.

Done once already (2026-06-11) for Moonshot AI's "IDC机房采购专家（智算中心方向）" —
see that page in this repo as the reference output.

## Standing setup (already in place — do not redo)

- Public repo `CherylWu3/job-posting-archives`, local clone at `~/job-posting-archives`.
- GitHub Pages enabled: branch `main`, path `/`, serving at `https://cherylwu3.github.io/job-posting-archives/`.
- `gh` CLI authenticated as CherylWu3 with `repo` scope.
- Note: the shell cannot `ls` `~/Desktop` (macOS TCC), but reading/writing individual
  files there by full path works fine.

## Step 1 — Get the authoritative source record

For MokaHR-hosted postings (URL shape `https://app.mokahr.com/apply/<org>/<siteId>#/job/<jobId>`):

1. The public API serves the verbatim record even after the job is delisted:
   `curl -s "https://api.mokahr.com/api-platform/v1/jobs/<org>/<jobId>"`
   Key fields: `title`, `description` (rich-text HTML — this is the verbatim JD),
   `commitment` (全职), `zhineng.name` (职位类别), `department.name`, `publishedAt`,
   `updatedAt`, `status`, `locations`. Save the JSON.
2. If verifying an existing draft: strip tags from the API `description`, split into
   lines, and confirm every non-empty line appears verbatim in the draft (Python).
3. Check whether the job is still listed: fetch the portal page **with a cookie jar**
   (it self-redirects on first request):
   `curl -sL -c /tmp/c.txt -b /tmp/c.txt "<portal URL without #fragment>" -A "Mozilla/5.0 ..."`.
   Parse the hidden `<input id="init-data" value="...">` (HTML-unescape, then JSON).
   `jobs` = currently listed postings; absence there + API `status: open` means
   "delisted but record still served".

For Feishu Hire portals (URL shape `https://<tenant>.jobs.feishu.cn/index/position/<id>/detail`,
e.g. MiniMax at tenant `vrfi1sk8a0`):

1. The job record API: `https://<tenant>.jobs.feishu.cn/api/v1/job/posts/<id>` →
   `data.job_post_detail` with `title`, `description` and `requirement` (both **plain
   text** with newlines, not HTML), `job_category.name`, `recruit_type` (parent =
   社招/校招, self = 全职), `city_list`, `publish_time` (epoch ms).
2. Portal branding is embedded in the detail page HTML (fetch with a browser UA):
   search for `"web_ui_config"` → `theme_color` (MiniMax: `#de1f84`), `slogan`
   (MiniMax: `MiniMax社招`), `navigation_bar_img_url` (logo), `tenant_name`.
3. Page template differences from Moka: white nav (logo + slogan, menus 首页/职位列表/
   校招), grey tag chips for 社招/全职/city/category, theme-color apply button
   (radius 6px, label 投递简历), sections 职位描述 + 职位要求 rendered in a
   `white-space: pre-line` div with the raw text embedded unmodified (verify both are
   exact substrings of the final file), footer "本招聘网站由 飞书招聘 提供招聘管理服务".
4. Only include facts from the record/portal config — don't add outside knowledge
   (e.g. legal entity names) to the page.

For other platforms: fetch the page, identify any underlying JSON/API the content comes
from, and treat that as the authoritative record; otherwise use the rendered HTML text.

## Step 2 — Build the styled, self-contained archive page

Replicate the original site's look using its real theme config, not guesswork.
For Moka sites, the config is in that same `init-data` JSON:

- `org.webSettings.nav` / `.footer` — nav/footer colors, logo URLs, company/ICP lines.
- `org.jobDetailSettings` — job detail page tokens. Moonshot/Kimi values used:
  page bg `#F4F6FB`; white cards radius `16px`; apply button bg `#0066FF`, white text,
  radius `300px` (pill); section titles with 4×16px `#0066FF` prefix bar; nav & footer
  bg `#141933`; secondary text `#89909E`; heading color `#141933`; body `#292C32`.
- Download the logo and favicon from their CDN and embed as **base64 data URIs** so the
  single HTML file has zero external dependencies.

Page structure (single HTML file, lang zh-CN, system font stack incl. PingFang SC):

1. Slim amber archival banner at the very top: states it's an archived copy, the
   archive date, that styling reproduces the original site, and that the JD text is
   verbatim from the API.
2. Dark nav (`#141933`): white logo (height 36px), white "社会招聘" pill label, menu
   items 首页 / 关于我们 / 社招职位 / 校招&实习职位 as inert text.
3. Two-column layout (max-width 1200px) on `#F4F6FB`:
   - Main card: job title (24px bold), meta row `全职 | 运营类 | 发布于 YYYY-MM-DD`,
     blue pill 申请 button linking to the original posting URL; then 职位描述 section
     with the API `description` HTML **embedded byte-for-byte unmodified**.
   - Sidebar cards: 职位信息 (职位性质/职位类别/所属部门/发布时间), 公司信息
     (favicon + company name), and an amber-bordered 存档信息 card (Job ID, Published,
     Last Updated / Closed, API status + delisted note, archive date, original URL,
     API endpoint).
4. Amber citation card under the main card: BibTeX (`@misc`, with the job ID and any
   closed date in `note`, `url` = the API endpoint) + Chicago style.
5. Dark footer: white logo, `© 北京月之暗面科技有限公司`, 京公网安备/京ICP lines.

**Verification (required):** the exact API `description` string must be a substring of
the final file — `desc in open(file).read()` must be `True`. No translations or edits
inside the JD; annotations live only in the clearly-marked amber blocks.

Filename convention: `<company>_<role-slug>_job_<published-date>.html`.

## Step 3 — Publish via GitHub Pages

In `~/job-posting-archives`:

1. Copy the new HTML file in.
2. Add a row to the README table and a list item to `index.html` (company, position,
   posted date, archived date, link).
3. Commit and push to `main` (Pages auto-deploys; no action needed).
4. Poll until live: `until curl -s -o /dev/null -w '%{http_code}' <pages-url> | grep -q 200; ...`
   (updates to an existing site deploy in well under a minute; use a background task,
   not foreground sleep).

## Step 4 — Report to the user

Provide the GitHub Pages URL — that is the citation link. No Wayback Machine or
archive.today submission (the user decided on 2026-06-11 that the GitHub Pages copy
alone is sufficient; do not re-suggest external archiving services).
