# Bitcoin Deposits Protocol - Single Page Site

## What We've Set Up

✅ **Single-page tufte-jekyll site**
- Focused on your Bitcoin Deposits protocol document
- Clean, academic-style presentation
- Custom Liquid tags for Tufte-style formatting
- Responsive design with sidenotes and margin notes
- MathJax support for mathematical expressions

✅ **Simplified structure**
- Single `index.md` with your protocol document
- No blog functionality or multiple pages
- Clean header with just site title
- Focused on content presentation

## Files Structure

### Core Theme Files
- `_layouts/` - Page layouts (default, full-width)
- `_includes/` - Reusable components (head, header, footer)
- `_plugins/` - Custom Liquid tags (sidenote, marginnote, newthought, etc.)
- `css/tufte.scss` - Main stylesheet
- `_scss/_settings.scss` - Theme customization variables

### Configuration
- `_config.yml` - Jekyll configuration for GitHub Pages
- `_data/options.yml` - Theme options (MathJax, fonts)
- `_data/social.yml` - Footer social links
- `Gemfile` - Ruby dependencies for GitHub Pages

### Content
- `index.md` - Your Bitcoin Deposits protocol document (only page)

## GitHub Pages Deployment

1. **Commit and push all files to your repository**
2. **Enable GitHub Pages in repository settings:**
   - Go to repository Settings → Pages
   - Set Source to "Deploy from a branch"
   - Set Branch to "main" (or "master")
   - Set folder to "/ (root)"
3. **Your site will be available at:** `https://ynniv.github.io/deposits/`

## Customization Options

### Colors and Fonts
Edit `_scss/_settings.scss` to customize:
- Typography (fonts)
- Colors (background, text, links)
- Layout (content width, margins)

### Features
Edit `_data/options.yml` to enable/disable:
- MathJax (set to `false` to disable)
- Lato font loading (set to `false` to use system fonts)

### Navigation
Edit `_includes/navigation.html` to modify site navigation

## Tufte-Jekyll Features You Can Use

### Custom Liquid Tags
- `{% newthought "Text" %}` - Small caps for section beginnings
- `{% sidenote "id" "note text" %}` - Numbered margin notes
- `{% marginnote "id" "note text" %}` - Unnumbered margin notes
- `{% epigraph "quote" "author" "source" %}` - Styled quotations
- `{% maincolumn "image.png" "caption" %}` - Images in main column
- `{% fullwidth "image.png" "caption" %}` - Full-width images
- `{% marginfigure "id" "image.png" "caption" %}` - Images in margin

### Mathematics
Use MathJax with double dollar signs:
```
$$ x = {-b \pm \sqrt{b^2-4ac} \over 2a} $$
```

## Next Steps

1. Commit and push the changes to GitHub
2. Enable GitHub Pages in repository settings
3. Wait a few minutes for the site to build
4. Visit your site and verify everything works
5. Continue editing your content using the Tufte-Jekyll tags

Your Bitcoin Deposits protocol document is now ready to be published as an elegant, academic-style website!