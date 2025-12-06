# Codebase Issues and Fix Instructions

**Generated:** 2025-12-06
**Repository:** loganthomas.github.io
**Branch:** cleanup

---

## Table of Contents
1. [Critical Issues](#critical-issues)
2. [High Priority Issues](#high-priority-issues)
3. [Medium Priority Issues](#medium-priority-issues)
4. [Low Priority Issues](#low-priority-issues)

---

## Critical Issues

### 1. Missing Include Files

**Problem:** Layout templates reference include files that don't exist. If config variables are set, Jekyll will fail.

**Files Affected:**
- `_layouts/post.html:29` references `disqus_comments.html`
- `_layouts/tutorial.html:29` references `disqus_comments.html`
- `_layouts/talk.html:29` references `disqus_comments.html`
- `_includes/head.html:10` references `google-analytics.html`

**Fix Instructions:**

**Option A:** Remove the references (if features not needed)
1. Open `_layouts/post.html`
2. Delete lines 28-30:
   ```liquid
   {%- if site.disqus.shortname -%}
     {%- include disqus_comments.html -%}
   {%- endif -%}
   ```
3. Repeat for `_layouts/tutorial.html:28-30` and `_layouts/talk.html:28-30`
4. Open `_includes/head.html`
5. Delete lines 9-11:
   ```liquid
   {%- if jekyll.environment == 'production' and site.google_analytics -%}
     {%- include google-analytics.html -%}
   {%- endif -%}
   ```

**Option B:** Create the missing files

For Disqus comments:
1. Create `_includes/disqus_comments.html`
2. Add Disqus embed code:
   ```html
   <div id="disqus_thread"></div>
   <script>
     var disqus_config = function () {
       this.page.url = '{{ page.url | absolute_url }}';
       this.page.identifier = '{{ page.url | absolute_url }}';
     };
     (function() {
       var d = document, s = d.createElement('script');
       s.src = 'https://{{ site.disqus.shortname }}.disqus.com/embed.js';
       s.setAttribute('data-timestamp', +new Date());
       (d.head || d.body).appendChild(s);
     })();
   </script>
   <noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
   ```
3. Add to `_config.yml`:
   ```yaml
   disqus:
     shortname: YOUR_DISQUS_SHORTNAME
   ```

For Google Analytics:
1. Create `_includes/google-analytics.html`
2. Add GA4 code:
   ```html
   <script async src="https://www.googletagmanager.com/gtag/js?id={{ site.google_analytics }}"></script>
   <script>
     window.dataLayer = window.dataLayer || [];
     function gtag(){dataLayer.push(arguments);}
     gtag('js', new Date());
     gtag('config', '{{ site.google_analytics }}');
   </script>
   ```
3. Add to `_config.yml`:
   ```yaml
   google_analytics: G-XXXXXXXXXX  # Your GA4 measurement ID
   ```

---

## High Priority Issues

### 2. Misspelled CSS Class Name

**Problem:** CSS class `highligher-rouge` is misspelled (should be `highlighter-rouge`)

**Files Affected:**
- `_layouts/post.html:19`
- `_layouts/talk.html:19`
- `_layouts/tutorial.html:19`

**Fix Instructions:**
1. Open `_layouts/post.html`
2. Find line 19:
   ```html
   <a href="/tags/{{ tag_name }}"><code class="highligher-rouge"><nobr>{{ tag_name }}</nobr></code>&nbsp;</a>
   ```
3. Replace with:
   ```html
   <a href="/tags/{{ tag_name }}"><code class="highlighter-rouge"><nobr>{{ tag_name }}</nobr></code>&nbsp;</a>
   ```
4. Repeat for `_layouts/talk.html:19`
5. Repeat for `_layouts/tutorial.html:19`

### 3. Malformed HTML Class Attribute

**Problem:** Footer has duplicate class prefix `footer-col-footer-col-3`

**Files Affected:**
- `_includes/footer.html:27`

**Fix Instructions:**
1. Open `_includes/footer.html`
2. Find line 27:
   ```html
   <div class="footer-col-footer-col-3">
   ```
3. Replace with:
   ```html
   <div class="footer-col footer-col-3">
   ```

---

## Medium Priority Issues

### 4. Inconsistent Archive Page Permalinks

**Problem:** Some archive pages missing explicit `permalink` in frontmatter, causing inconsistent URL structures.

**Files Affected:**
- `archive/2015.md` - missing permalink
- `archive/2016.md` - missing permalink
- `archive/2020.md` - missing permalink
- `archive/2021.md` - missing permalink
- `archive/2022.md` - missing permalink
- `archive/2023.md` - missing permalink

**Fix Instructions:**

For each file, add `permalink` to frontmatter:

**archive/2015.md:**
```yaml
---
layout: page
title: 2015 Turkey Bowl Results
permalink: /turkey-bowl/archive/2015/
---
```

**archive/2016.md:**
```yaml
---
layout: page
title: 2016 Turkey Bowl Results
permalink: /turkey-bowl/archive/2016/
---
```

**archive/2020.md:**
```yaml
---
layout: page
title: 2020 Turkey Bowl Results
permalink: /turkey-bowl/archive/2020/
---
```

**archive/2021.md:**
```yaml
---
layout: page
title: 2021 Turkey Bowl Results
permalink: /turkey-bowl/archive/2021/
---
```

**archive/2022.md:**
```yaml
---
layout: page
title: 2022 Turkey Bowl Results
permalink: /turkey-bowl/archive/2022/
---
```

**archive/2023.md:**
```yaml
---
layout: page
title: 2023 Turkey Bowl Results
permalink: /turkey-bowl/archive/2023/
---
```

### 5. Undefined Configuration Variables

**Problem:** Templates reference config variables not defined in `_config.yml`

**Variables Used But Not Defined:**
- `site.google_analytics` (used in `_includes/head.html:9`)
- `site.disqus.shortname` (used in multiple layouts)
- `site.show_excerpts` (used in `_includes/profile_content.html:60` and `_layouts/blog_home.html:25`)

**Fix Instructions:**

Add to `_config.yml` (after line 32):
```yaml
# Comments (optional - only add if using Disqus)
# disqus:
#   shortname: YOUR_SHORTNAME_HERE

# Analytics (optional - only add if using Google Analytics)
# google_analytics: G-XXXXXXXXXX

# Show post excerpts on listing pages
show_excerpts: false
```

### 6. Dead Code in Social Icons

**Problem:** 20+ lines of commented-out social media icons in `_includes/social.html`

**Files Affected:**
- `_includes/social.html:12-27, 33-34`

**Fix Instructions:**
1. Open `_includes/social.html`
2. Remove commented-out sections (lines 12-27 and 33-34)
3. Keep only the active social links (GitHub, LinkedIn, Twitter/X)

Alternatively, if you want to keep for reference:
1. Create `_includes/social.html.bak` as backup
2. Clean up the main file

### 7. Non-HTTPS URL

**Problem:** One external link uses HTTP instead of HTTPS

**Files Affected:**
- `blog/_posts/2024-06-12-pytorch-layer-output-dims.md`

**Fix Instructions:**
1. Open `blog/_posts/2024-06-12-pytorch-layer-output-dims.md`
2. Search for `http://yann.lecun.com/exdb/mnist/`
3. Replace with `https://yann.lecun.com/exdb/mnist/` (if HTTPS is available)
4. Or note that this is an external reference and may not support HTTPS

---

## Low Priority Issues

### 8. Tracked System Files

**Problem:** macOS `.DS_Store` files are tracked in git (should be ignored)

**Files Affected:**
- `assets/.DS_Store`
- `assets/images/.DS_Store`
- `assets/images/2018/.DS_Store`
- `assets/images/2019/.DS_Store`
- `assets/images/2020/.DS_Store`
- `assets/logs/.DS_Store`

**Fix Instructions:**
1. Remove from git tracking:
   ```bash
   git rm --cached assets/.DS_Store
   git rm --cached assets/images/.DS_Store
   git rm --cached assets/images/2018/.DS_Store
   git rm --cached assets/images/2019/.DS_Store
   git rm --cached assets/images/2020/.DS_Store
   git rm --cached assets/logs/.DS_Store
   ```
2. Verify `.gitignore` contains `.DS_Store` (it already does)
3. Commit the removal:
   ```bash
   git commit -m "Remove tracked .DS_Store files"
   ```

### 9. Outdated Dependencies

**Problem:** Ruby gems and Bundler version from 2023

**Files Affected:**
- `Gemfile.lock` (last updated 2023-07-25)

**Fix Instructions:**
1. Update dependencies:
   ```bash
   bundle update
   ```
2. Test site builds correctly:
   ```bash
   bundle exec jekyll serve
   ```
3. If successful, commit updated `Gemfile.lock`:
   ```bash
   git add Gemfile.lock
   git commit -m "Update Ruby dependencies"
   ```

---

## Fix Order Recommendations

**Suggested execution order:**

1. **First:** Fix critical issues (#1)
   - Decide: remove references OR create missing files
   - Prevents potential build failures

2. **Second:** Fix high priority issues (#2, #3)
   - Quick fixes, immediate visual impact
   - Fix misspelled CSS class
   - Fix malformed HTML class

3. **Third:** Fix medium priority issues (#4, #5, #6, #7)
   - Add missing permalinks to archive pages
   - Document config variables
   - Clean up dead code
   - Fix HTTP URL

4. **Fourth:** Fix low priority issues (#8, #9)
   - Remove .DS_Store files
   - Update dependencies

---

## Testing After Fixes

After making fixes, test the site:

```bash
# Clean build
bundle exec jekyll clean

# Build and serve locally
bundle exec jekyll serve --host 0.0.0.0

# Check for build errors
# Visit http://localhost:4000
# Test navigation, blog posts, archive pages
# Verify no 404s in console
```

---

## Notes

- All fixes should be made on the `cleanup` branch
- Test thoroughly before merging to `main`
- Consider creating backups of files before major changes
- The `favicon.ico` and `_includes/profile_content.html` issues have already been fixed

---

## Positive Aspects (No Action Needed)

The following are already in good shape:
- Well-organized file structure
- Consistent frontmatter across all markdown files
- Good use of Jekyll includes for DRY principles
- No exposed credentials or security issues
- All referenced images exist and are accessible
- Proper gitignore configuration
