# Automated Supabase Reports with GitHub Actions

This repository contains an automated workflow that generates and publishes reports from Supabase data using R Markdown and GitHub Actions. The system runs on a schedule and can also be triggered manually, providing a seamless way to create regular data reports.

## Architecture Overview

The workflow orchestrates several key components:
- **Supabase**: Source database that provides the data for analysis
- **R Markdown**: Template defining the report structure and analysis logic
- **GitHub Actions**: Automation engine that handles scheduling and execution
- **GitHub Pages**: Hosting platform for the published reports

## Workflow Components

### 1. GitHub Actions Workflow

The primary automation is defined in `.github/workflows/supabase-auto-reports.yml`:

```yaml
name: Generate and Deploy Supabase Report

on: 
  schedule:
    - cron: '0 8 * * 1' # i.e every Monday @ 8AM UTC
  workflow_dispatch: # Allow manual trigger
  push:
    branches:
      - main  # Triggers on pushes to main branch

permissions:
  contents: write
  pages: write
  id-token: write
  
jobs:
  render:
    runs-on: ubuntu-latest
    env:
      SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
      SUPABASE_KEY: ${{ secrets.SUPABASE_KEY }}
      
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        
      - name: Set Up R
        uses: r-lib/actions/setup-r@v2
        with:
          r-version: '4.4.2'
          
      - name: Setup Pandoc
        uses: r-lib/actions/setup-pandoc@v2
          
      - name: Install system dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y libcurl4-openssl-dev libssl-dev libxml2-dev

      - name: Install R dependencies
        run: |
          R -e 'options(repos = c(CRAN = "https://cloud.r-project.org"))'
          R -e 'install.packages(c("rmarkdown", "knitr"), dependencies=TRUE)'
          R -e 'install.packages(c("httr", "jsonlite", "dplyr", "ggplot2", "lubridate"), dependencies=TRUE)'
          R -e 'install.packages(c("tinytex", "DBI", "RPostgres"), dependencies=TRUE)'
          R -e 'tinytex::install_tinytex()'

      - name: Verify R package installation
        run: |
          R -e 'installed_packages <- installed.packages()[,"Package"]; cat("Installed packages:", paste(installed_packages, collapse=", "), "\n")'

      - name: Render RMarkdown
        run: |
          Rscript -e 'rmarkdown::render("auto-report.Rmd", output_file = "report.pdf")'
        
      - name: Check Query Status
        id: query_check
        run: |
          if [ -f query_status.txt ] && [ "$(cat query_status.txt)" == "SUCCESS" ]; then
            echo "Query executed successfully, proceeding with deployment."
            echo "query_status=success" >> $GITHUB_OUTPUT
          else
            echo "Query failed, aborting deployment."
            if [ -f error_log.txt ]; then
              echo "Error details:"
              cat error_log.txt
            fi
            echo "query_status=failed" >> $GITHUB_OUTPUT
            exit 1
          fi
          
      - name: Deploy PDF
        if: steps.query_check.outputs.query_status == 'success'
        uses: actions/upload-artifact@v4
        with: 
          name: supabase-report-pdf
          path: report.pdf
          
      - name: Deploy to Github Pages
        if: steps.query_check.outputs.query_status == 'success'
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: .
          publish_branch: gh-pages
          keep_files: true
```

### 2. R Markdown Template (`auto-report.Rmd`)

The R Markdown document defines the report structure and contains the code to:
1. Connect to Supabase and retrieve data
2. Process and analyze the data
3. Generate visualizations and insights
4. Format everything into a PDF report

The template uses the following structure:

```r
---
title: "Automated Supabase Report"
date: "`r format(Sys.time(), '%B %d, %Y')`"
output: pdf_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = FALSE, warning = FALSE, message = FALSE)

# Load required libraries
library(DBI)
library(httr)
library(jsonlite)
library(dplyr)
library(ggplot2)
library(lubridate)

# Supabase connection parameters from environment variables
supabase_url <- Sys.getenv("SUPABASE_URL")
supabase_key <- Sys.getenv("SUPABASE_KEY")

# Your data retrieval and processing code here
# ...

# Write query status for GitHub Actions to check
writeLines("SUCCESS", "query_status.txt")
```

## Security Configuration

This workflow uses GitHub Secrets to securely store sensitive credentials:
- `SUPABASE_URL`: The endpoint URL for your Supabase instance
- `SUPABASE_KEY`: The API key used to authenticate with Supabase

These are securely injected into the workflow environment without being exposed in logs.

## Execution Flow

1. **Trigger**: The workflow runs either:
   - On schedule (every Monday at 8 AM UTC)
   - When manually triggered through the GitHub Actions UI
   - When changes are pushed to the main branch

2. **Environment Setup**:
   - Ubuntu runner is provisioned
   - R and required system dependencies are installed
   - Pandoc is set up for document conversion
   - R packages are installed

3. **Report Generation**:
   - The R Markdown document is rendered to PDF
   - The rendering process connects to Supabase, fetches data, and creates the report

4. **Status Check**:
   - The workflow checks for a "SUCCESS" string in the `query_status.txt` file
   - If the file contains "SUCCESS", deployment proceeds
   - If not, the workflow fails and logs the error

5. **Deployment**:
   - The generated PDF is saved as a workflow artifact
   - The report and related files are published to GitHub Pages
   - Previous files are preserved on the gh-pages branch

## Error Handling

The workflow includes several error handling mechanisms:
- Verifies required dependencies are installed
- Checks for successful query execution
- Logs errors to aid troubleshooting
- Conditionally deploys only on success

## Viewing Reports

Reports can be accessed in two ways:
1. As workflow artifacts in the GitHub Actions run
2. On the GitHub Pages site for the repository at: `https://github.com/[username]/[repository-name]/blob/gh-pages/report.pdf`
in my case it's: `https://github.com/AkanimohOD19A/automated-Rmd-outputs/blob/gh-pages/report.pdf`

 


## Local Development

To develop and test locally:

1. Clone the repository
2. Create a `.Renviron` file with:
   ```
   SUPABASE_URL=your_supabase_url
   SUPABASE_KEY=your_supabase_key
   ```
3. Install required R packages
4. Run `rmarkdown::render("auto-report.Rmd")` to generate the report

## Customization

To customize this workflow:
1. Modify `auto-report.Rmd` to change report content and analysis
2. Edit the GitHub Actions workflow to adjust scheduling or dependencies
3. Update the deployment mechanism if different output formats are needed

## Troubleshooting

Common issues and solutions:
- **Missing R packages**: Update the dependency installation step
- **Pandoc errors**: Ensure the correct version of Pandoc is installed
- **Database connection failures**: Check Supabase credentials and network access
- **PDF generation issues**: Look for LaTeX-related errors in the logs

## Requirements

- GitHub repository with Actions enabled
- GitHub Pages configured for the repository
- Supabase instance with appropriate tables and permissions
- R (version 4.0+) with required packages

## License

*MIT License*