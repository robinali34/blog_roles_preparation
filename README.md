# Robina Li's Roles Preparation Blog

A blog focused on interview preparation, role breakdowns, and career development resources. Powered by Jekyll and GitHub Pages.

## Features

- Clean, responsive design
- Markdown-based blog posts
- Automatic GitHub Pages deployment
- RSS feed support
- Social media integration
- Syntax highlighting for code blocks

## Getting Started

### Prerequisites

- Ruby (version 3.1 or higher)
- Bundler gem

### Local Development

1. Clone this repository
2. Install dependencies:
   ```bash
   bundle install
   ```
3. Serve the site locally:
   ```bash
   bundle exec jekyll serve
   ```
4. Open your browser to `http://localhost:4000`

### Adding New Posts

1. Create a new file in the `_posts` directory
2. Name it with the format: `YYYY-MM-DD-your-post-title.md`
3. Add front matter at the top:
   ```yaml
   ---
   layout: post
   title: "Your Post Title"
   date: YYYY-MM-DD HH:MM:SS -0000
   categories: interview-preparation career
   ---
   ```
4. Write your content in Markdown below the front matter

### Deployment

This blog is automatically deployed to GitHub Pages when you push changes to the `main` branch. The deployment is handled by GitHub Actions.

**Repository**: `blog_roles_preparation`  
**Live URL**: `https://robinali34.github.io/blog_roles_preparation/`

## Customization

### Site Configuration

Edit `_config.yml` to customize:
- Site title and description
- Author information
- Social media links
- Plugins and themes

### Styling

- Main stylesheet: `assets/main.scss`
- Layouts: `_layouts/`
- Includes: `_includes/`

### Pages

Create new pages by adding `.md` files to the root directory with front matter:
```yaml
---
layout: page
title: "Page Title"
permalink: /page-url/
---
```

## License

This project is open source and available under the [MIT License](LICENSE).

